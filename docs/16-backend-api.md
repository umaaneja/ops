# Backend API — Ticket Agent System
# docs/16-backend-api.md

---

## Overview

REST API + SSE streaming + WebSocket for approvals.
All endpoints require authentication. All responses are JSON.
Base URL: https://api.yourapp.com/v1

Authentication: Bearer token in Authorization header.
Tenant is resolved from the token — no tenant_id in URLs.

---

## 1. Authentication

```
POST   /auth/token           Exchange API key for JWT
POST   /auth/refresh         Refresh expiring JWT
DELETE /auth/token           Revoke token
```

```typescript
// POST /auth/token
// Request
{ "api_key": "sk-..." }

// Response 200
{
  "access_token": "eyJ...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "tenant_id": "uuid",
  "scopes": ["runs:write", "runs:read", "approvals:write"]
}
```

---

## 2. Tickets — Ingest

```
POST   /tickets              Submit a new ticket (creates a run)
GET    /tickets/:id/run      Get the current run for a ticket
```

```typescript
// POST /tickets
// Request
{
  "ticket_id":      "JIRA-4821",            // required — external ticket ID
  "source":         "jira",                 // required — 'jira'|'servicenow'|'zendesk'|'pagerduty'|'manual'
  "title":          "API Latency Spike",    // required
  "description":    "string",              // required — full ticket body
  "priority":       "P1",                  // required — P1|P2|P3|P4
  "reporter_email": "user@company.com",    // optional
  "reporter_slack": "U12345",              // optional — Slack user ID
  "affected_services": ["user-api"],       // optional — hint to triage agent
  "metadata": {}                           // optional — arbitrary key/value
}

// Response 201
{
  "run_id":      "uuid",
  "ticket_id":   "JIRA-4821",
  "state":       "IDLE",
  "created_at":  "2024-01-15T23:12:16Z",
  "events_url":  "/v1/runs/uuid/events",   // SSE stream URL
  "websocket_url": "wss://api.yourapp.com/v1/runs/uuid/ws"
}

// POST /tickets (webhook from Jira/ServiceNow/PagerDuty)
// Same endpoint — source system sends webhook payload
// Adapter normalizes to common format before processing
// Returns same 201 response
```

---

## 3. Runs

```
GET    /runs                  List runs (paginated)
GET    /runs/:id              Get run detail
DELETE /runs/:id              Stop a run
POST   /runs/:id/pause        Pause a running run
POST   /runs/:id/resume       Resume a paused run
POST   /runs/:id/skip-step    Skip current step (triggers replan)
GET    /runs/:id/plan         Get current plan (all versions)
GET    /runs/:id/steps        List all steps for a run
GET    /runs/:id/steps/:stepId Get step detail with tool call
GET    /runs/:id/audit        Get full audit trace for a run
GET    /runs/:id/cost         Get cost breakdown for a run
GET    /runs/:id/workspace    List workspace files for a run
GET    /runs/:id/events       SSE stream (see section 7)
```

```typescript
// GET /runs
// Query params: state, priority, ticket_id, from, to, limit, cursor
// Response 200
{
  "runs": [
    {
      "run_id":         "uuid",
      "ticket_id":      "JIRA-4821",
      "state":          "EXECUTING",
      "priority":       "P1",
      "step_index":     6,
      "total_steps":    10,
      "cost_usd":       0.24,
      "max_cost_usd":   2.00,
      "active_agent":   "DIAGNOSIS",
      "started_at":     "2024-01-15T23:12:16Z",
      "updated_at":     "2024-01-15T23:16:41Z",
      "was_skillless":  false
    }
  ],
  "cursor": "next_page_cursor",
  "total": 142
}

// GET /runs/:id
// Response 200
{
  "run_id":           "uuid",
  "ticket_id":        "JIRA-4821",
  "ticket_source":    "jira",
  "state":            "EXECUTING",
  "previous_state":   "DISPATCHING",
  "state_change_reason": "Step 5 dispatched to Resolution Agent",

  "plan_version":     2,
  "plan":             { /* current plan JSON */ },
  "step_index":       6,
  "total_steps":      10,
  "active_agent":     "RESOLUTION",

  "cost_usd":         0.24,
  "max_cost_usd":     2.00,
  "cost_warned":      false,

  "skill_set":        ["db-connection-management", "api-latency-diagnosis"],
  "was_skillless":    false,

  "hypothesis": {
    "text":      "pgbouncer connection pool exhausted",
    "confidence": 0.83,
    "alternatives": [
      { "text": "upstream slow queries", "dead": true },
      { "text": "network partition",     "dead": true }
    ]
  },

  "started_at":     "2024-01-15T23:12:16Z",
  "deadline_at":    "2024-01-16T00:12:16Z",
  "updated_at":     "2024-01-15T23:16:41Z",

  "harness_config_version": "1.4.2",
  "attempt_number": 1
}

// DELETE /runs/:id (Stop)
// Request body optional
{ "reason": "string" }
// Response 200
{ "run_id": "uuid", "state": "INTERRUPTED", "stopped_at": "timestamp" }

// POST /runs/:id/pause
// Response 200
{ "run_id": "uuid", "paused": true }

// POST /runs/:id/resume
// Response 200
{ "run_id": "uuid", "paused": false, "state": "EXECUTING" }

// POST /runs/:id/skip-step
// Response 200
{ "run_id": "uuid", "skipped_step": 6, "message": "Step skipped — replan triggered" }

// GET /runs/:id/plan
// Response 200
{
  "current_version": 2,
  "versions": [
    {
      "version": 1,
      "created_at": "timestamp",
      "created_reason": "initial",
      "steps": [ /* array of plan steps */ ]
    },
    {
      "version": 2,
      "created_at": "timestamp",
      "created_reason": "Step 3 failed: SSH timeout — switching to CloudWatch API",
      "steps": [ /* revised steps */ ]
    }
  ]
}

// GET /runs/:id/steps
// Response 200
{
  "steps": [
    {
      "step_id":      "uuid",
      "step_index":   1,
      "agent_role":   "DIAGNOSIS",
      "tool_name":    "run_command",
      "command_id":   "pg_connection_count",
      "status":       "SUCCESS",
      "cost_usd":     0.003,
      "latency_ms":   1124,
      "started_at":   "timestamp",
      "ended_at":     "timestamp",
      "output_summary": "cl_active=100 (MAX) cl_waiting=892 maxwait=31.2s"
    }
  ]
}

// GET /runs/:id/cost
// Response 200
{
  "total_usd": 0.34,
  "budget_usd": 2.00,
  "budget_used_pct": 17,
  "breakdown": {
    "by_agent": {
      "TRIAGE":      0.04,
      "DIAGNOSIS":   0.18,
      "RESOLUTION":  0.08,
      "VERIFICATION": 0.02,
      "POST_INCIDENT": 0.02
    },
    "by_type": {
      "model_calls": 0.28,
      "tool_calls":  0.04,
      "kb_queries":  0.02
    },
    "by_step": [ /* per step */ ]
  }
}

// GET /runs/:id/workspace
// Response 200
{
  "files": [
    {
      "path":       "reasoning/step_01.json",
      "size_bytes": 1240,
      "type":       "scratchpad",
      "created_at": "timestamp"
    },
    {
      "path":       "diagnostics/combined.json",
      "size_bytes": 8420,
      "type":       "diagnostic",
      "created_at": "timestamp"
    }
  ]
}
```

---

## 4. Approvals

```
GET    /approvals              List pending approvals for tenant
GET    /approvals/:id          Get approval detail
POST   /approvals/:id/approve  Approve action
POST   /approvals/:id/reject   Reject action
GET    /runs/:id/approvals     List all approvals for a run
```

```typescript
// GET /approvals
// Query params: status=PENDING, run_id
// Response 200
{
  "approvals": [
    {
      "approval_id":    "uuid",
      "run_id":         "uuid",
      "ticket_id":      "JIRA-4821",
      "status":         "PENDING",
      "action_description": "Reload pgbouncer connection pool on db-primary-01",
      "tool_name":      "run_command",
      "command_id":     "pgbouncer_reload",
      "risk_level":     "MEDIUM",
      "agent_confidence": 0.83,
      "confidence_threshold": 0.80,
      "rollback_available": true,
      "rollback_description": "pgbouncer restart will undo the reload effect",
      "timeout_at":     "timestamp",
      "created_at":     "timestamp"
    }
  ]
}

// GET /approvals/:id
// Response 200 — full payload for display in approval UI
{
  "approval_id":        "uuid",
  "run_id":             "uuid",
  "ticket_id":          "JIRA-4821",
  "status":             "PENDING",

  "action_description": "Reload pgbouncer connection pool on db-primary-01",
  "tool_name":          "run_command",
  "command_id":         "pgbouncer_reload",
  "payload": {
    "command":  "pgbouncer -R /var/run/pgbouncer/pgbouncer.sock",
    "host":     "db-primary-01 (display name)",
    "service":  "payment-service",
    "env":      "production"
  },

  "risk_level":           "MEDIUM",
  "agent_confidence":     0.83,
  "confidence_threshold": 0.80,
  "confidence_met":       true,

  "expected_outcome":   "Connection pool clears, cl_waiting drops to 0 within 30s",
  "expected_downtime":  "~30 seconds of connection interruption",
  "data_loss_risk":     false,

  "rollback_available": true,
  "rollback_description": "Run pgbouncer_restart to undo — 2-3 min downtime",
  "rollback_command_id": "pgbouncer_restart",

  "run_context": {
    "active_hypothesis": "pgbouncer pool exhausted",
    "hypothesis_confidence": 0.83,
    "steps_completed": 6,
    "steps_total": 10,
    "cost_so_far": 0.24
  },

  "timeout_at":   "timestamp",
  "created_at":   "timestamp"
}

// POST /approvals/:id/approve
// Request
{ "reason": "optional string" }
// Response 200
{
  "approval_id":  "uuid",
  "status":       "APPROVED",
  "decided_by":   "user@company.com",
  "decided_at":   "timestamp",
  "run_resumed":  true
}

// POST /approvals/:id/reject
// Request
{ "reason": "optional string" }
// Response 200
{
  "approval_id":  "uuid",
  "status":       "REJECTED",
  "decided_by":   "user@company.com",
  "decided_at":   "timestamp",
  "replan_triggered": true
}
```

---

## 5. Audit & Observability

```
GET    /runs/:id/audit              Full audit trace for a run
GET    /runs/:id/audit/:event_id    Single audit event detail
GET    /runs/:id/reasoning          Scratchpad history for a run
GET    /runs/:id/reasoning/:step    Single scratchpad
GET    /policy/decisions            Policy decision log (paginated)
GET    /policy/decisions/:id        Single policy decision detail
```

```typescript
// GET /runs/:id/audit
// Response 200
{
  "run_id": "uuid",
  "events": [
    {
      "audit_id":     "uuid",
      "event_type":   "POLICY_DECISION",
      "agent_role":   "DIAGNOSIS",
      "tool_name":    "run_command",
      "decision":     "ALLOWED",
      "gate_fired":   null,
      "reason":       "All 16 policy gates passed",
      "risk_level":   "READ_ONLY",
      "confidence":   null,
      "cost_at_time": 0.13,
      "created_at":   "timestamp"
    },
    {
      "audit_id":     "uuid",
      "event_type":   "POLICY_DECISION",
      "agent_role":   "RESOLUTION",
      "tool_name":    "run_command",
      "decision":     "APPROVAL_REQUIRED",
      "gate_fired":   "gate_14_requires_approval",
      "reason":       "pgbouncer_reload requires human approval (risk: MEDIUM)",
      "risk_level":   "MEDIUM",
      "confidence":   0.83,
      "cost_at_time": 0.24,
      "created_at":   "timestamp"
    },
    {
      "audit_id":     "uuid",
      "event_type":   "HUMAN_APPROVAL",
      "decision":     "APPROVED",
      "decided_by":   "oncall@company.com",
      "created_at":   "timestamp"
    },
    {
      "audit_id":     "uuid",
      "event_type":   "COMMAND_EXECUTED",
      "tool_name":    "run_command",
      "command_id":   "pgbouncer_reload",
      "host":         "db-primary-01",
      "exit_code":    0,
      "latency_ms":   1311,
      "created_at":   "timestamp"
    }
  ]
}

// GET /runs/:id/reasoning
// Response 200
{
  "steps": [
    {
      "step_index":        1,
      "what_i_know":       "Pool is exhausted...",
      "what_i_dont_know":  "Root cause of sudden spike...",
      "active_hypothesis": "connection_pool_exhaustion",
      "confidence":        0.72,
      "hypotheses_count":  3,
      "dead_ends":         [],
      "created_at":        "timestamp"
    }
  ]
}
```

---

## 6. Knowledge Base & Episodes

```
GET    /knowledge/search          Search KB (for UI / debugging)
GET    /knowledge/gaps            List knowledge gap reports
GET    /knowledge/gaps/:id        Get gap detail
GET    /knowledge/drafts/kb       List KB document drafts
GET    /knowledge/drafts/kb/:id   Get KB draft
POST   /knowledge/drafts/kb/:id/approve  Approve and publish KB draft
POST   /knowledge/drafts/kb/:id/reject   Reject KB draft
GET    /knowledge/drafts/skills   List skill drafts
POST   /knowledge/drafts/skills/:id/approve  Approve and publish skill
POST   /knowledge/drafts/skills/:id/reject

GET    /episodes                  Search past incidents
GET    /episodes/:id              Get episode detail
GET    /episodes/:id/steps        Get step-by-step detail for episode
```

```typescript
// GET /knowledge/search?q=pgbouncer+connection+pool&service=payment-service
// Response 200
{
  "results": [
    {
      "chunk_id":   "string",
      "title":      "PostgreSQL Connection Pool Management",
      "content":    "string (first 300 chars)",
      "source_type": "runbook",
      "relevance":  0.91,
      "confidence": 0.85,
      "service":    "payment-service",
      "last_validated": "2024-11-01"
    }
  ],
  "total": 12,
  "query_time_ms": 48
}

// GET /episodes?q=api+latency&service=payment-service&limit=5
// Response 200
{
  "episodes": [
    {
      "episode_id":       "uuid",
      "ticket_id":        "JIRA-4201",
      "similarity":       0.89,
      "ticket_summary":   "string",
      "root_cause":       "connection_pool_exhaustion",
      "resolution_text":  "string",
      "outcome":          "RESOLVED",
      "resolution_mins":  18,
      "was_skillless":    false,
      "created_at":       "timestamp"
    }
  ]
}

// POST /knowledge/drafts/kb/:id/approve
// Request
{ "notes": "reviewed and verified — published to production KB" }
// Response 200
{
  "draft_id":     "uuid",
  "status":       "PUBLISHED",
  "published_at": "timestamp",
  "chunk_ids":    ["kb-chunk-1", "kb-chunk-2"]  // created in Elasticsearch
}
```

---

## 7. SSE Event Stream

Connect to GET /runs/:id/events for real-time updates.
The browser EventSource API handles reconnection automatically.
On reconnect, send the Last-Event-ID header to get missed events.

```typescript
// Client-side connection
const es = new EventSource(`/v1/runs/${runId}/events`, {
  headers: { Authorization: `Bearer ${token}` }
});

es.onmessage = (e) => {
  const event = JSON.parse(e.data);
  handleEvent(event);
};
```

### Event types and payloads

```typescript
// STATE_CHANGED
{
  type: "STATE_CHANGED",
  run_id: "uuid",
  from_state: "DISPATCHING",
  to_state: "EXECUTING",
  reason: "Step 4 dispatched to Resolution Agent",
  ts: "timestamp"
}

// AGENT_SPAWNED
{
  type: "AGENT_SPAWNED",
  run_id: "uuid",
  agent_id: "uuid",
  agent_role: "DIAGNOSIS",
  skills_loaded: ["db-connection-management"],
  ts: "timestamp"
}

// HYPOTHESIS_UPDATED
{
  type: "HYPOTHESIS_UPDATED",
  run_id: "uuid",
  hypothesis_text: "pgbouncer connection pool exhausted",
  confidence: 0.83,
  delta: 0.11,
  alternatives: [
    { text: "upstream slow queries", dead: true },
    { text: "network partition", dead: true }
  ],
  ts: "timestamp"
}

// COMMAND_STARTED
{
  type: "COMMAND_STARTED",
  run_id: "uuid",
  step_id: "uuid",
  agent_role: "DIAGNOSIS",
  tool_name: "run_command",
  command_id: "pg_connection_count",
  display_command: "psql ... -c \"SHOW POOLS;\"",  // sanitized for display
  server_display: "db-primary-01",                 // sanitized host
  ts: "timestamp"
}

// COMMAND_OUTPUT
{
  type: "COMMAND_OUTPUT",
  run_id: "uuid",
  step_id: "uuid",
  line: " payments  |   100/100 |        892 |   31.2s",
  stream: "stdout",   // or "stderr"
  ts: "timestamp"
}

// COMMAND_COMPLETED
{
  type: "COMMAND_COMPLETED",
  run_id: "uuid",
  step_id: "uuid",
  exit_code: 0,
  latency_ms: 1124,
  success: true,
  output_summary: "cl_active=100 (MAX) cl_waiting=892 maxwait=31.2s",
  ts: "timestamp"
}

// POLICY_DECISION
{
  type: "POLICY_DECISION",
  run_id: "uuid",
  tool_name: "run_command",
  command_id: "pgbouncer_reload",
  decision: "APPROVAL_REQUIRED",
  risk_level: "MEDIUM",
  confidence: 0.83,
  reason: "pgbouncer_reload requires human approval",
  ts: "timestamp"
}

// APPROVAL_REQUIRED
{
  type: "APPROVAL_REQUIRED",
  run_id: "uuid",
  approval_id: "uuid",
  action_description: "Reload pgbouncer connection pool",
  command_id: "pgbouncer_reload",
  risk_level: "MEDIUM",
  confidence: 0.83,
  timeout_at: "timestamp",
  ts: "timestamp"
}

// APPROVAL_DECISION
{
  type: "APPROVAL_DECISION",
  run_id: "uuid",
  approval_id: "uuid",
  decision: "APPROVED",
  decided_by: "oncall@company.com",
  ts: "timestamp"
}

// REPLAN_TRIGGERED
{
  type: "REPLAN_TRIGGERED",
  run_id: "uuid",
  trigger: "STEP_FAILURE",
  failed_step: 3,
  reason: "SSH timeout to app-server-03 after 3 retries",
  old_plan_version: 1,
  new_plan_version: 2,
  changes: {
    dropped_steps: [3],
    new_steps: [
      { step_index: "3b", tool: "run_command", command_id: "cloudwatch_logs_query" }
    ]
  },
  ts: "timestamp"
}

// COST_UPDATE
{
  type: "COST_UPDATE",
  run_id: "uuid",
  cost_usd: 0.24,
  budget_usd: 2.00,
  budget_pct: 12,
  ts: "timestamp"
}

// STEP_COMPLETED
{
  type: "STEP_COMPLETED",
  run_id: "uuid",
  step_id: "uuid",
  step_index: 5,
  agent_role: "DIAGNOSIS",
  tool_name: "run_command",
  status: "SUCCESS",
  cost_usd: 0.003,
  ts: "timestamp"
}

// VERIFICATION_RESULT
{
  type: "VERIFICATION_RESULT",
  run_id: "uuid",
  step_index: 6,
  confirmed: true,
  confidence: 0.96,
  summary: "cl_waiting=0 · pool recovered",
  ts: "timestamp"
}

// RUN_COMPLETED
{
  type: "RUN_COMPLETED",
  run_id: "uuid",
  outcome: "RESOLVED",
  resolution_summary: "pgbouncer connection pool exhausted — resolved by reload",
  total_cost_usd: 0.34,
  total_duration_ms: 52000,
  steps_executed: 10,
  replan_count: 1,
  ts: "timestamp"
}

// AGENT_LOG
{
  type: "AGENT_LOG",
  run_id: "uuid",
  agent_role: "DIAGNOSIS",
  log_level: "info",    // info | warn | error | success | section
  message: "string",
  ts: "timestamp"
}

// RUN_RESUMED (after crash recovery)
{
  type: "RUN_RESUMED",
  run_id: "uuid",
  from_step: 4,
  reason: "crash_recovery",
  ts: "timestamp"
}
```

---

## 8. Internal APIs (not exposed to end users)

These are called by workers, the watchdog, and scheduled jobs.
Protected by internal service auth, not user JWT.

```
POST   /internal/runs/:id/resume          Resume after crash (watchdog)
POST   /internal/runs/:id/state           Transition state (orchestrator only)
POST   /internal/runs/:id/cost            Increment cost (workers)
POST   /internal/graph-updates            Queue graph update job (knowledge creator)
POST   /internal/kb-search-stats         Update KB chunk usage stats
POST   /internal/eval/run                 Run offline eval suite
GET    /internal/watchdog/stuck-runs      Get runs to recover (watchdog)
POST   /internal/circuit-breakers/:tool/record-failure
POST   /internal/circuit-breakers/:tool/record-success
GET    /internal/circuit-breakers        Get all circuit breaker states
```

---

## 9. Webhooks — Inbound

For receiving events from external systems.

```
POST   /webhooks/tickets/jira            Jira issue webhook
POST   /webhooks/tickets/servicenow      ServiceNow incident webhook
POST   /webhooks/tickets/pagerduty       PagerDuty alert webhook
POST   /webhooks/tickets/zendesk         Zendesk ticket webhook
POST   /webhooks/approvals/slack         Slack Approve/Reject button click
POST   /webhooks/approvals/email         Email-based approval response
```

All webhook endpoints:
- Verify signature (HMAC) before processing
- Return 200 immediately (async processing via queue)
- Are idempotent (duplicate webhooks safe)

---

## 10. Error Responses

All errors follow the same schema:

```typescript
// 4xx / 5xx
{
  "error": {
    "code":    "POLICY_BLOCKED",           // machine-readable
    "message": "Agent is not permitted to call execute_runbook_step",
    "details": { /* additional context */ },
    "run_id":  "uuid",                     // if relevant
    "trace_id": "uuid"                     // for support
  }
}

// Common error codes:
// UNAUTHORIZED              missing or invalid token
// FORBIDDEN                 valid token but insufficient scope
// NOT_FOUND                 run_id / approval_id does not exist
// CONFLICT                  e.g. approving an already-decided approval
// POLICY_BLOCKED            policy guard rejected the action
// BUDGET_EXCEEDED           run has hit cost ceiling
// RATE_LIMITED              too many requests
// VALIDATION_ERROR          request body schema violation
// RUN_NOT_ACTIVE            trying to control a completed run
// CIRCUIT_OPEN              target tool's circuit breaker is open
// INTERNAL_ERROR            unexpected server error
```

---

## 11. Rate Limits (API)

```
Endpoint group                    Limit
────────────────────────────────────────────────────────
POST /tickets                     100 / minute per tenant
GET  /runs                        300 / minute per tenant
GET  /runs/:id/events (SSE)       50 concurrent connections per tenant
POST /approvals/:id/approve       60 / minute per user
GET  /knowledge/search            200 / minute per tenant
POST /internal/*                  1000 / minute (internal service)
```

Headers returned on every response:
```
X-RateLimit-Limit:     100
X-RateLimit-Remaining: 87
X-RateLimit-Reset:     1705362000
```

---

## 12. Upload — Ticket Attachments and Context Files

Users can upload files to attach to a ticket run (e.g. a log dump, a config file,
a screenshot). These are available to agents via the filesystem tool.

```
POST   /runs/:id/uploads         Upload a file (multipart/form-data)
GET    /runs/:id/uploads         List uploads for a run
GET    /runs/:id/uploads/:fileId Download a file
DELETE /runs/:id/uploads/:fileId Delete a file
```

```typescript
// POST /runs/:id/uploads
// Content-Type: multipart/form-data
// Body fields:
//   file: File (required)
//   description: string (optional — hint to agent about what this file contains)
//   agent_accessible: boolean (default true — if false, only humans can see it)

// Response 201
{
  "file_id":          "uuid",
  "filename":         "app-logs-2024-01-15.txt",
  "size_bytes":       241840,
  "content_type":     "text/plain",
  "description":      "Application logs from the time of incident",
  "workspace_path":   "uploads/app-logs-2024-01-15.txt",
  "agent_accessible": true,
  "created_at":       "timestamp"
}

// Limits
// Max file size:          50 MB
// Max files per run:      20
// Allowed content types:
//   text/plain, text/csv, application/json, application/xml,
//   image/png, image/jpeg (for screenshots)
//   application/zip (will be unzipped, files inspected)

// Files are available to agents via:
//   read_from_filesystem tool with path = "uploads/{filename}"
// Agent is notified of uploads via AGENT_LOG event on the SSE stream:
//   "New file uploaded by user: uploads/app-logs-2024-01-15.txt
//    Description: Application logs from the time of incident"
```

---

## 13. BUILDPLAN Note for Claude Code

Implement the API in this order:

Phase 11a (before UI):
1. POST /tickets — ingest and create run
2. GET  /runs/:id — read run state
3. GET  /runs/:id/events — SSE stream (stubbed events first)
4. POST /approvals/:id/approve and /reject
5. DELETE /runs/:id (stop)

Phase 11b (with UI):
6. GET /runs — list with pagination
7. GET /runs/:id/steps and /audit
8. GET /runs/:id/plan (all versions)
9. GET /runs/:id/reasoning
10. POST /runs/:id/pause, /resume, /skip-step

Phase 11c (knowledge systems):
11. GET /knowledge/search
12. GET /episodes
13. GET /knowledge/gaps and /drafts
14. POST /knowledge/drafts/*/approve and /reject

Phase 11d (uploads):
15. POST/GET /runs/:id/uploads
16. Webhook inbound endpoints

Middleware required on all routes:
- Auth middleware (verify JWT, extract tenant_id)
- Request ID middleware (generate trace_id)
- Rate limit middleware
- Request validation middleware (zod schemas)
- Error handler middleware (formats all errors to standard schema)
- Audit logging middleware (logs every request to audit_requests table)
