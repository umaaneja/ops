# Crash Recovery & Restart — Ticket Agent System
# docs/15-crash-recovery.md

---

## Mental Model

Every run must be resumable from any point of failure.
This is not optional — a P1 incident agent that crashes mid-resolution
and cannot restart is worse than no agent at all.

The system achieves this through one principle:
  All state lives outside the process.

The process (the Node.js worker) is stateless and disposable.
It reads state from PostgreSQL + Redis on startup and continues from where it left off.
Crashing and restarting is designed to be indistinguishable from a brief pause.

---

## 1. What Can Crash and Where State Lives

```
Component           Can crash?   State lives in         Recovery approach
─────────────────────────────────────────────────────────────────────────────────
Orchestrator        YES          PostgreSQL (agent_runs) Resume from last state
Agent worker        YES          Redis + filesystem      Re-read context, continue
Model call          YES          Nowhere yet             Retry the call
Tool execution      YES          Redis (idempotency)     Re-check cache, re-run if safe
SSE stream          YES          Redis (event stream)    Client reconnects, replays
Approval wait       YES          PostgreSQL (approvals)  Check approval status on resume
UI session          YES          Server-side SSE         Client reconnects
MCP gateway         YES          Stateless               Retry with circuit breaker
Neo4j               YES          Persisted               Graph update job retries
Elasticsearch       YES          Persisted               KB writes idempotent
```

---

## 2. Orchestrator Crash Recovery

### How it detects a crash

A separate watchdog process (or Kubernetes liveness probe) checks for stuck runs.
A run is "stuck" when:

```sql
-- Runs that are in an active state but have not been updated in > 2 minutes
SELECT run_id, state, updated_at, active_agent
FROM agent_runs
WHERE state IN ('PLANNING', 'DISPATCHING', 'EXECUTING', 'REPLANNING')
  AND updated_at < NOW() - INTERVAL '2 minutes'
  AND tenant_id = $tenant_id;
```

### Recovery sequence on stuck run detection

```typescript
async function recoverStuckRun(runId: string): Promise<void> {

  const run = await db.query(
    'SELECT * FROM agent_runs WHERE run_id = $1 FOR UPDATE',
    [runId]
  );

  // Step 1 — check if still stuck (not just slow)
  if (run.updated_at > Date.now() - 120_000) return; // recovered on its own

  // Step 2 — check if there's an agent actually running (heartbeat key in Redis)
  const heartbeat = await redis.get(`run:${runId}:heartbeat`);
  if (heartbeat) return; // agent is alive, just slow — check again in 60s

  // Step 3 — identify where to resume
  const lastCompletedStep = await db.query(
    `SELECT step_index FROM agent_steps
     WHERE run_id = $1 AND status = 'SUCCESS'
     ORDER BY step_index DESC LIMIT 1`,
    [runId]
  );

  const resumeFromStep = lastCompletedStep
    ? lastCompletedStep.step_index + 1
    : 0;

  // Step 4 — check if there is a pending approval
  const pendingApproval = await db.query(
    `SELECT approval_id FROM approvals
     WHERE run_id = $1 AND status = 'PENDING'`,
    [runId]
  );

  if (pendingApproval) {
    // Run was waiting for approval when it crashed.
    // Check if the approval was given while we were down.
    const approval = await db.query(
      'SELECT status FROM approvals WHERE approval_id = $1',
      [pendingApproval.approval_id]
    );
    if (approval.status === 'APPROVED') {
      // Approval came in while crashed — resume with approval
      await resumeRun(runId, resumeFromStep, { approval_granted: true });
    } else if (approval.status === 'PENDING') {
      // Still waiting — re-send the approval notification (idempotent)
      await resendApprovalNotification(pendingApproval.approval_id);
      // Do not resume — wait for human
    }
    return;
  }

  // Step 5 — check if the step in progress used a non-idempotent tool
  const inProgressStep = await db.query(
    `SELECT * FROM agent_steps
     WHERE run_id = $1 AND status = 'RUNNING'`,
    [runId]
  );

  if (inProgressStep) {
    // Was a write tool in progress when we crashed?
    const toolMeta = await toolRegistry.get(inProgressStep.tool_name);

    if (toolMeta.is_idempotent) {
      // Safe to re-run — idempotency key will return cached result if it completed
      await resumeRun(runId, inProgressStep.step_index);
    } else if (toolMeta.risk_level === 'READ_ONLY') {
      // Read-only — always safe to re-run
      await resumeRun(runId, inProgressStep.step_index);
    } else {
      // Non-idempotent write was in progress — need to verify state before re-running
      await markStepForVerification(inProgressStep.step_id);
      await transitionState(runId, 'REPLANNING', 'Crash during non-idempotent write — verifying state before resume');
      await spawnVerificationAgent(runId, inProgressStep);
    }
    return;
  }

  // Step 6 — clean resume from last known good step
  await resumeRun(runId, resumeFromStep);
}

async function resumeRun(runId: string, fromStep: number, context?: object) {
  // Update heartbeat immediately so watchdog does not double-recover
  await redis.set(`run:${runId}:heartbeat`, '1', 'EX', 30);

  // Rebuild context from memory layers
  const workingMemory = await redis.getAll(`run:${runId}:*`);
  const filesystemIndex = JSON.parse(workingMemory[`run:${runId}:filesystem:index`] || '[]');
  const investigation = workingMemory[`run:${runId}:investigation:summary`];
  const hypothesis = JSON.parse(workingMemory[`run:${runId}:hypothesis:active`] || '{}');

  // If Redis was wiped (worst case), reconstruct from PostgreSQL
  if (!workingMemory || Object.keys(workingMemory).length === 0) {
    await reconstructWorkingMemoryFromPostgres(runId);
  }

  // Re-spawn the correct agent from current state
  await orchestrator.dispatch(runId, fromStep, context);

  // Emit resume event to SSE stream
  await emitEvent(runId, 'RUN_RESUMED', {
    from_step: fromStep,
    reason: 'crash_recovery',
    ...context
  });
}
```

---

## 3. Heartbeat System

Every active agent sends a heartbeat to Redis every 15 seconds.
The watchdog checks for missing heartbeats every 30 seconds.

```typescript
// In every agent worker loop
class AgentHeartbeat {
  private interval: NodeJS.Timeout;

  start(runId: string, agentId: string) {
    this.interval = setInterval(async () => {
      await redis.set(
        `run:${runId}:heartbeat`,
        JSON.stringify({ agent_id: agentId, ts: Date.now() }),
        'EX', 30  // expires in 30s — if agent dies, key expires
      );
      // Also update agent_runs.updated_at so watchdog SQL query works
      await db.query(
        'UPDATE agent_runs SET updated_at = NOW() WHERE run_id = $1',
        [runId]
      );
    }, 15_000);
  }

  stop() {
    clearInterval(this.interval);
  }
}
```

---

## 4. Redis Crash — Working Memory Loss

Redis is the most dangerous crash because it holds all in-flight run state.
Mitigate with:

**Primary: Redis Sentinel or Redis Cluster**
Use Redis Sentinel (3 nodes) for automatic failover. Recovery time < 30 seconds.
Working memory survives as long as one replica is alive.

**Fallback: Reconstruct from PostgreSQL**
If Redis is completely lost, the system can reconstruct working memory:

```typescript
async function reconstructWorkingMemoryFromPostgres(runId: string): Promise<void> {
  const run = await db.query('SELECT * FROM agent_runs WHERE run_id = $1', [runId]);
  const steps = await db.query(
    'SELECT * FROM agent_steps WHERE run_id = $1 ORDER BY step_index ASC',
    [runId]
  );

  // Rebuild meta
  await redis.hset(`run:${runId}:meta`, {
    state: run.state,
    plan_version: run.plan_version,
    step_index: run.step_index,
    cost_usd: run.cost_usd,
  });

  // Rebuild step results from what was persisted
  for (const step of steps) {
    if (step.status === 'SUCCESS' && step.output_json) {
      await redis.set(
        `run:${runId}:step:${step.step_id}:result`,
        JSON.stringify(step.output_json)
      );
    }
  }

  // Rebuild plan
  await redis.set(
    `run:${runId}:plan:current`,
    JSON.stringify(run.plan_json)
  );

  // NOTE: scratchpads and investigation summary are on the filesystem.
  // Rebuild investigation summary by reading the latest step's scratchpad.
  const latestScratchpad = await s3.getObject(
    `workspace/runs/${runId}/reasoning/step_${run.step_index - 1}.json`
  );
  if (latestScratchpad) {
    const scratchpad = JSON.parse(latestScratchpad);
    await redis.set(
      `run:${runId}:investigation:summary`,
      scratchpad.what_i_know?.slice(0, 200) || 'Investigation context lost — resuming'
    );
    await redis.set(
      `run:${runId}:hypothesis:active`,
      JSON.stringify({
        hypothesis_text: scratchpad.active_hypothesis || 'Unknown — see scratchpad',
        confidence: scratchpad.hypotheses?.[0]?.confidence || 0,
      })
    );
  }
}
```

**What is lost and cannot be reconstructed:**
- Cached graph and KB query results (10-min TTL cache) — re-queried on resume, slight latency cost
- SSE stream history — client UI reconnects and shows "Agent resumed after crash"
- In-flight model call — retried from scratch, idempotent

---

## 5. Filesystem / S3 Crash

S3 is durable by design (11 nines). If writing to local filesystem instead of S3,
use a mounted EFS volume (NFS-backed, survives instance crashes).

If a write was in progress when the crash happened:

```typescript
// All filesystem writes use atomic write pattern:
// 1. Write to .tmp file
// 2. Rename to final name (atomic on POSIX, idempotent on S3)
async function writeWorkspaceFile(path: string, content: string): Promise<void> {
  const tmpPath = path + '.tmp.' + Date.now();
  await s3.put(tmpPath, content);
  await s3.copy(tmpPath, path);     // copy is atomic in S3
  await s3.delete(tmpPath);
}

// On resume, check for .tmp files and clean them up
async function cleanupOrphanedTmpFiles(runId: string): Promise<void> {
  const orphans = await s3.list(`workspace/runs/${runId}/`, { suffix: '.tmp.' });
  for (const f of orphans) await s3.delete(f);
}
```

---

## 6. Model Call Crash

If the process crashes during a model call, the call is simply lost.
No state was written yet (model calls produce output, not side effects).
On resume, the orchestrator detects the step is still PENDING and re-dispatches.

```typescript
// Model calls are wrapped in a try-finally that always cleans up
async function callModel(runId: string, stepId: string, prompt: object): Promise<ModelOutput> {
  await db.query(
    "UPDATE agent_steps SET status = 'RUNNING', started_at = NOW() WHERE step_id = $1",
    [stepId]
  );

  try {
    const result = await anthropic.messages.create(prompt);
    await db.query(
      "UPDATE agent_steps SET status = 'SUCCESS', output_json = $2 WHERE step_id = $1",
      [stepId, result]
    );
    return result;
  } catch (error) {
    await db.query(
      "UPDATE agent_steps SET status = 'FAILED', error_message = $2 WHERE step_id = $1",
      [stepId, error.message]
    );
    throw error;
  }
}
// If the process crashes inside the try block:
// - step stays in RUNNING state
// - watchdog detects it has been RUNNING for > 2 minutes
// - watchdog marks it FAILED and triggers re-dispatch
```

---

## 7. Tool Execution Crash

This is the most dangerous crash. A write tool was executing and we do not know
if it completed before the crash.

The idempotency key in Redis protects against double-execution for most tools.
But if Redis also crashed, the idempotency key is gone.

```typescript
async function executeToolSafely(
  callId: string,
  tool: string,
  input: object,
  context: ExecutionContext
): Promise<ToolResult> {

  // Check idempotency cache
  const cached = await redis.get(`idem:${callId}`);
  if (cached) {
    return JSON.parse(cached); // already ran, return cached result
  }

  // Check PostgreSQL idempotency (fallback if Redis lost)
  const dbCached = await db.query(
    'SELECT output_json FROM tool_calls WHERE call_id = $1 AND success = true',
    [callId]
  );
  if (dbCached) {
    // Re-populate Redis cache
    await redis.set(`idem:${callId}`, JSON.stringify(dbCached.output_json), 'EX', 86400);
    return dbCached.output_json;
  }

  // Not cached — execute
  const result = await mcpGateway.execute(tool, input, context);

  // Write to both PostgreSQL and Redis atomically
  await db.query(
    `INSERT INTO tool_calls (call_id, run_id, step_id, tool_name, input_hash, output_json, success, latency_ms)
     VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
     ON CONFLICT (call_id) DO NOTHING`,
    [callId, runId, stepId, tool, hash(input), result, true, result.latency_ms]
  );
  await redis.set(`idem:${callId}`, JSON.stringify(result), 'EX', 86400);

  return result;
}
```

**For write tools that cannot be safely re-run (non-idempotent):**
The Verification Agent is spawned on resume to check if the action completed.
Its job is to observe system state and determine if the tool's effect is present.

```typescript
// Example: pgbouncer_reload crash recovery
async function verifyPgbouncerReloadEffect(service: string): Promise<'COMPLETED' | 'NOT_COMPLETED'> {
  // Check if connection pool is healthy (the effect of the reload)
  const result = await runCommand('pg_connection_count', {}, context);
  const clWaiting = result.data.cl_waiting;
  const maxWait = result.data.maxwait_ms;

  if (clWaiting < 10 && maxWait < 1000) {
    return 'COMPLETED'; // reload happened, pool is healthy
  }
  return 'NOT_COMPLETED'; // still exhausted, reload did not complete
}
```

---

## 8. Database Crash (PostgreSQL)

Use PostgreSQL with streaming replication and automatic failover (e.g. Patroni, AWS RDS Multi-AZ).
Failover time: 30–60 seconds.

During failover:
- Active runs are paused (heartbeat fails, watchdog detects)
- Approval webhooks queue in the SSE stream
- On reconnect, watchdog scans for stuck runs and recovers them

All critical writes use transactions:

```typescript
// State transitions are always in a transaction
async function transitionState(runId: string, newState: RunState, reason: string) {
  await db.transaction(async (tx) => {
    const run = await tx.query(
      'SELECT state FROM agent_runs WHERE run_id = $1 FOR UPDATE',
      [runId]
    );
    validateTransition(run.state, newState); // throws if invalid
    await tx.query(
      `UPDATE agent_runs
       SET state = $2, previous_state = state, state_changed_at = NOW(),
           state_change_reason = $3, updated_at = NOW()
       WHERE run_id = $1`,
      [runId, newState, reason]
    );
    await tx.query(
      `INSERT INTO policy_audit (run_id, tenant_id, tool_name, decision, reason, agent_role)
       VALUES ($1, $2, 'state_transition', 'ALLOWED', $3, $4)`,
      [runId, tenantId, `${run.state} → ${newState}: ${reason}`, currentAgent]
    );
  });
}
```

---

## 9. Kubernetes Crash Recovery

If running on Kubernetes, the watchdog is a separate Deployment (not a sidecar).

```yaml
# k8s/watchdog-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-watchdog
spec:
  replicas: 1                    # singleton — only one watchdog at a time
  strategy:
    type: Recreate               # not RollingUpdate — prevents two watchdogs
  template:
    spec:
      containers:
        - name: watchdog
          image: ticket-agent-watchdog:latest
          env:
            - name: SCAN_INTERVAL_SECONDS
              value: "30"
            - name: STUCK_THRESHOLD_SECONDS
              value: "120"
          livenessProbe:
            exec:
              command: ["node", "healthcheck.js"]
            periodSeconds: 30
```

Agent workers run as a Deployment with horizontal scaling:

```yaml
# k8s/agent-worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-worker
spec:
  replicas: 5                    # 5 workers, each handles multiple runs
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2
  template:
    spec:
      containers:
        - name: worker
          image: ticket-agent-worker:latest
          lifecycle:
            preStop:
              exec:
                # On graceful shutdown, finish current step then stop
                command: ["node", "scripts/graceful-shutdown.js"]
          terminationGracePeriodSeconds: 120   # 2 minutes to finish current step
```

Graceful shutdown sequence:

```typescript
// scripts/graceful-shutdown.js
process.on('SIGTERM', async () => {
  console.log('SIGTERM received — graceful shutdown starting');

  // Stop accepting new runs
  await redis.set('worker:accepting_runs', '0');

  // Wait for current steps to complete (max 90 seconds)
  const deadline = Date.now() + 90_000;
  while (Date.now() < deadline) {
    const activeRuns = await getActiveRunsForThisWorker();
    if (activeRuns.length === 0) break;

    // Check if current steps are at a safe boundary
    const allAtBoundary = activeRuns.every(r => r.state === 'DISPATCHING');
    if (allAtBoundary) break;

    await sleep(1000);
  }

  // Mark any still-running steps as INTERRUPTED
  // (watchdog will recover them)
  await markActiveStepsInterrupted();

  console.log('Graceful shutdown complete');
  process.exit(0);
});
```

---

## 10. Approval Webhook Crash Recovery

When a human clicks Approve/Reject in Slack, the webhook must not be lost.

```typescript
// Approval webhooks go to a queue first, not directly to the run
// This decouples the Slack response from the agent process

// Slack → POST /webhooks/approvals → queue message → worker processes it
async function handleApprovalWebhook(req: Request): Promise<void> {
  const { approval_id, decision, decided_by } = req.body;

  // Write to DB immediately (durable)
  await db.query(
    `UPDATE approvals
     SET status = $2, decided_by = $3, decided_at = NOW()
     WHERE approval_id = $1`,
    [approval_id, decision, decided_by]
  );

  // Queue the resume signal (in case worker is down)
  await queue.add('resume_run_after_approval', {
    approval_id,
    decision,
    run_id: approval.run_id,
  }, {
    attempts: 5,
    backoff: { type: 'exponential', delay: 2000 }
  });

  // Acknowledge Slack immediately (do not wait for agent to resume)
  res.json({ ok: true });
}

// Worker processes the queue
queue.process('resume_run_after_approval', async (job) => {
  const { approval_id, decision, run_id } = job.data;

  // Check if run is already resumed (idempotent)
  const run = await db.query('SELECT state FROM agent_runs WHERE run_id = $1', [run_id]);
  if (!['EXECUTING', 'DISPATCHING'].includes(run.state)) {
    // Run is not waiting anymore — skip
    return;
  }

  // Resume the run
  await redis.set(`run:${run_id}:approval:${approval_id}:status`, decision);
  await orchestrator.resumeFromApproval(run_id, approval_id, decision);
});
```

---

## 11. SSE Stream Reconnection

When the UI client loses connection and reconnects:

```typescript
// Server: SSE endpoint with replay
app.get('/runs/:runId/events', async (req, res) => {
  const { runId } = req.params;
  const lastEventId = req.headers['last-event-id'];  // browser sends this on reconnect

  // Set SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Replay missed events if reconnecting
  if (lastEventId) {
    const missedEvents = await redis.xrange(
      `stream:${runId}:events`,
      lastEventId,   // from last seen event
      '+'            // to latest
    );
    for (const event of missedEvents) {
      res.write(`id: ${event.id}\ndata: ${event.data}\n\n`);
    }
  }

  // Forward new events
  const unsubscribe = eventBus.subscribe(runId, (event) => {
    res.write(`id: ${event.id}\ndata: ${JSON.stringify(event)}\n\n`);
  });

  req.on('close', unsubscribe);
});

// Client: auto-reconnect with EventSource
const es = new EventSource(`/runs/${runId}/events`);
es.onerror = () => {
  // EventSource reconnects automatically with last-event-id
  // Show "reconnecting..." in UI
};
```

---

## 12. Crash Recovery Runbook (for humans)

When a run is stuck and the watchdog has not recovered it:

```bash
# 1. Check run state
psql $DATABASE_URL -c "
  SELECT run_id, state, step_index, cost_usd, updated_at,
         NOW() - updated_at AS stuck_for
  FROM agent_runs
  WHERE run_id = 'YOUR_RUN_ID';"

# 2. Check last steps
psql $DATABASE_URL -c "
  SELECT step_index, tool_name, status, error_message, started_at, ended_at
  FROM agent_steps
  WHERE run_id = 'YOUR_RUN_ID'
  ORDER BY step_index DESC
  LIMIT 10;"

# 3. Check Redis working memory
redis-cli KEYS "run:YOUR_RUN_ID:*"

# 4. Check filesystem workspace
aws s3 ls s3://YOUR_BUCKET/workspace/runs/YOUR_RUN_ID/ --recursive

# 5. Force resume from specific step (use with caution)
curl -X POST https://api.yourapp.com/internal/runs/YOUR_RUN_ID/resume \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"from_step": 4, "reason": "manual recovery by oncall"}'

# 6. Force stop if unrecoverable
curl -X POST https://api.yourapp.com/internal/runs/YOUR_RUN_ID/stop \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"reason": "manual stop — unrecoverable crash"}'
```
