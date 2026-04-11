# Policy Definitions — Ticket Agent System
# docs/14-policies.md

---

## Mental Model

Policies are not code scattered through the codebase.
They are data — stored in files and a database, loaded at runtime, evaluated by one
single PolicyGuard service. Nothing else enforces policy. Everything else trusts that
if PolicyGuard said allowed, it is allowed.

Policies live in three places:
  1. policy/             — YAML files checked into git (version controlled, human readable)
  2. policies table      — database (runtime overrides, tenant-specific rules)
  3. service_access_registry — per-service command whitelists and forbidden patterns

PolicyGuard loads all three on startup and merges them. Database rules override file
rules for the same key. This lets you hot-patch a policy without a deploy.

---

## 1. Policy File Structure

```
policy/
  base.yaml                   ← applies to all tenants, all agents
  authority-matrix.yaml       ← which agent role can call which tool
  risk-levels.yaml            ← risk classification per tool / command
  confidence-thresholds.yaml  ← minimum confidence per risk level
  budget-rules.yaml           ← cost ceilings and warning thresholds
  rate-limits.yaml            ← max calls per tool per run
  forbidden-patterns.yaml     ← shell patterns always blocked
  destructive-actions.yaml    ← actions that always need human approval
  tenants/
    {tenant_id}.yaml          ← tenant-specific overrides
```

---

## 2. base.yaml

```yaml
# policy/base.yaml
# Global rules that apply everywhere, always.
# Cannot be overridden by tenant files.

version: "1.0"

global:
  # Hard stops — these fire before any other rule
  always_block:
    - rule: no_credentials_in_input
      description: Reject any tool input containing patterns that look like credentials
      patterns:
        - '(?i)(password|passwd|secret|token|api_key|apikey)\s*[:=]\s*\S+'
        - 'AKIA[0-9A-Z]{16}'           # AWS access key
        - '[0-9a-f]{40}'               # potential SHA1 secret
      action: BLOCK
      reason: "Credential-like string detected in tool input"

    - rule: no_out_of_scope_tenant
      description: Tool inputs must only reference resources belonging to this run's tenant
      action: BLOCK
      reason: "Cross-tenant resource reference detected"

    - rule: no_shell_injection
      description: Reject inputs with shell metacharacters not in approved parameter positions
      patterns:
        - '[;&|`$](?!\(date)'         # common injection chars (allow $(date) for log queries)
        - '\$\((?!date)'              # command substitution except $(date)
        - '>\s*/dev/null'
        - '\|\s*bash'
        - '\|\s*sh'
        - 'wget.*\|'
        - 'curl.*\|.*sh'
      action: BLOCK
      reason: "Shell injection pattern detected"

  # Budget hard ceiling — no run can exceed this regardless of ticket priority
  absolute_max_cost_usd: 10.00

  # Max steps hard ceiling
  absolute_max_steps: 50

  # Approval timeout — if human does not respond, auto-escalate
  approval_timeout_mins: 15

  # Context window compaction trigger (% of model's max context)
  compaction_threshold_pct: 75

  # Max parallel agents per run
  max_parallel_agents: 5

  # Credential TTL hard limits
  read_credential_ttl_seconds: 3600      # 1 hour
  write_credential_ttl_seconds: 120      # 2 minutes
  admin_credential_ttl_seconds: 60       # 1 minute
```

---

## 3. authority-matrix.yaml

This is the most important policy file.
Every agent role has an explicit whitelist of allowed tools.
If a tool is not on the list, it is blocked. No exceptions.

```yaml
# policy/authority-matrix.yaml

version: "1.0"

# Format:
#   {agent_role}:
#     allowed_tools:
#       - tool_name
#     denied_tools:             # explicit deny (belt + suspenders)
#       - tool_name
#     notes: string

TRIAGE:
  notes: "Read-only. Classifies the ticket. Cannot touch external systems."
  allowed_tools:
    - get_ticket_details
    - search_past_incidents
  denied_tools:
    - "*"                       # deny everything not in allowed_tools
  max_model_calls: 2
  max_cost_usd: 0.05

DIAGNOSIS:
  notes: "Read-only. Investigates. Can run commands but only read-only ones."
  allowed_tools:
    - get_ticket_details
    - search_knowledge_base
    - search_past_incidents
    - run_command               # only read-only command_ids (enforced by risk-levels.yaml)
    - get_server_logs
    - read_from_filesystem
    - write_to_filesystem       # can write scratchpads and diagnostic outputs
    - query_monitoring_api      # Datadog, Grafana — read only
    - query_knowledge_graph
  denied_tools:
    - execute_runbook_step
    - update_ticket
    - notify_user
    - escalate
    - request_human_approval
    - delete_resource
  max_model_calls: 15
  max_cost_usd: 0.40
  max_loops: 10

RESOLUTION:
  notes: "Can execute write operations but ALL writes go through policy guard confidence check."
  allowed_tools:
    - run_command               # read-only AND write command_ids (gated by confidence)
    - execute_runbook_step
    - update_ticket
    - write_to_filesystem
    - read_from_filesystem
    - request_human_approval
    - notify_user
    - query_monitoring_api
  denied_tools:
    - delete_resource
    - drop_database
    - terminate_instance        # covered by destructive-actions.yaml separately
  max_model_calls: 20
  max_cost_usd: 0.60
  max_loops: 20

VERIFICATION:
  notes: "Read-only. Confirms resolution worked. Cannot execute anything."
  allowed_tools:
    - run_command               # only read-only command_ids
    - get_server_logs
    - query_monitoring_api
    - get_ticket_details
    - read_from_filesystem
    - write_to_filesystem       # writes verification results
  denied_tools:
    - execute_runbook_step
    - update_ticket
    - notify_user
    - escalate
    - request_human_approval
  max_model_calls: 4
  max_cost_usd: 0.10
  max_loops: 2

COMMUNICATION:
  notes: "Only sends messages. Cannot touch servers, databases, or tickets directly."
  allowed_tools:
    - notify_user
    - read_from_filesystem
    - write_to_filesystem
    - get_ticket_details        # read only, to get reporter contact info
    - query_knowledge_graph     # for team/person lookup (escalation routing)
  denied_tools:
    - run_command
    - execute_runbook_step
    - update_ticket
    - get_server_logs
    - escalate                  # must go through orchestrator, not directly
  max_model_calls: 3
  max_cost_usd: 0.05

POST_INCIDENT:
  notes: "Writes to memory systems only. Cannot touch live infrastructure."
  allowed_tools:
    - write_to_filesystem
    - read_from_filesystem
    - update_ticket             # to close the ticket with resolution note
    - write_episode             # internal tool, writes to episodes table
    - write_graph_updates       # internal tool, queues graph update job
    - write_skill_feedback      # internal tool, writes to knowledge_gap_reports
  denied_tools:
    - run_command
    - execute_runbook_step
    - get_server_logs
    - notify_user
    - request_human_approval
  max_model_calls: 4
  max_cost_usd: 0.08

KNOWLEDGE_CREATOR:
  notes: "Analysis and draft generation only. All output goes to review queue."
  allowed_tools:
    - read_from_filesystem
    - write_to_filesystem
    - read_episode
    - write_kb_draft            # internal tool, writes to kb_drafts (status=PENDING_REVIEW)
    - write_skill_draft         # internal tool, writes to skill_drafts (status=PENDING_REVIEW)
    - write_graph_updates
    - write_skill_feedback
  denied_tools:
    - run_command
    - execute_runbook_step
    - update_ticket
    - notify_user
    - publish_kb_document       # humans publish, never the agent
    - publish_skill             # humans publish, never the agent
  max_model_calls: 8
  max_cost_usd: 0.15
```

---

## 4. risk-levels.yaml

Every tool and every command_id gets a risk classification.
Risk level determines: confidence threshold, approval requirement, credential scope.

```yaml
# policy/risk-levels.yaml

version: "1.0"

tools:
  # Pure read — no server access
  get_ticket_details:       { risk: READ_ONLY,  requires_approval: false }
  search_knowledge_base:    { risk: READ_ONLY,  requires_approval: false }
  search_past_incidents:    { risk: READ_ONLY,  requires_approval: false }
  read_from_filesystem:     { risk: READ_ONLY,  requires_approval: false }
  query_monitoring_api:     { risk: READ_ONLY,  requires_approval: false }
  query_knowledge_graph:    { risk: READ_ONLY,  requires_approval: false }

  # Read with server access
  get_server_logs:          { risk: READ_ONLY,  requires_approval: false }

  # Write — internal only
  write_to_filesystem:      { risk: LOW,        requires_approval: false }
  write_episode:            { risk: LOW,        requires_approval: false }
  write_graph_updates:      { risk: LOW,        requires_approval: false }
  write_skill_feedback:     { risk: LOW,        requires_approval: false }
  write_kb_draft:           { risk: LOW,        requires_approval: false }
  write_skill_draft:        { risk: LOW,        requires_approval: false }

  # Write — external
  notify_user:              { risk: LOW,        requires_approval: false }
  update_ticket:            { risk: LOW,        requires_approval: false }

  # Write — server execution
  run_command:
    risk: VARIES              # determined by command_id, see commands section below
    requires_approval: VARIES

  execute_runbook_step:     { risk: HIGH,       requires_approval: true }
  request_human_approval:   { risk: READ_ONLY,  requires_approval: false }  # meta-tool

  # Escalation
  escalate:                 { risk: LOW,        requires_approval: false }

# Per command_id risk levels (used when tool = run_command)
commands:
  # Read-only diagnostics — no approval ever
  pg_connection_count:      { risk: READ_ONLY,  requires_approval: false, credential: read }
  pgbouncer_status:         { risk: READ_ONLY,  requires_approval: false, credential: read }
  pg_slow_queries:          { risk: READ_ONLY,  requires_approval: false, credential: read }
  pg_stat_activity:         { risk: READ_ONLY,  requires_approval: false, credential: read }
  tail_log:                 { risk: READ_ONLY,  requires_approval: false, credential: read }
  cloudwatch_logs_query:    { risk: READ_ONLY,  requires_approval: false, credential: read }
  check_disk_usage:         { risk: READ_ONLY,  requires_approval: false, credential: read }
  check_memory_usage:       { risk: READ_ONLY,  requires_approval: false, credential: read }
  check_process_list:       { risk: READ_ONLY,  requires_approval: false, credential: read }
  netstat_check:            { risk: READ_ONLY,  requires_approval: false, credential: read }
  k8s_pod_status:           { risk: READ_ONLY,  requires_approval: false, credential: read }
  k8s_pod_logs:             { risk: READ_ONLY,  requires_approval: false, credential: read }

  # Low risk writes — need confidence >= 0.75, no human approval
  clear_application_cache:  { risk: LOW,        requires_approval: false, credential: write, min_confidence: 0.75 }
  rotate_log_file:          { risk: LOW,        requires_approval: false, credential: write, min_confidence: 0.75 }
  flush_redis_key:          { risk: LOW,        requires_approval: false, credential: write, min_confidence: 0.75 }

  # Medium risk — need confidence >= 0.80 AND human approval
  pgbouncer_reload:         { risk: MEDIUM,     requires_approval: true,  credential: write, min_confidence: 0.80 }
  nginx_reload:             { risk: MEDIUM,     requires_approval: true,  credential: write, min_confidence: 0.80 }
  k8s_rollout_restart:      { risk: MEDIUM,     requires_approval: true,  credential: write, min_confidence: 0.80 }
  scale_deployment:         { risk: MEDIUM,     requires_approval: true,  credential: write, min_confidence: 0.80 }

  # High risk — need confidence >= 0.90 AND human approval
  pgbouncer_restart:        { risk: HIGH,       requires_approval: true,  credential: write, min_confidence: 0.90 }
  service_restart:          { risk: HIGH,       requires_approval: true,  credential: write, min_confidence: 0.90 }
  k8s_pod_delete:           { risk: HIGH,       requires_approval: true,  credential: write, min_confidence: 0.90 }
  change_pool_size:         { risk: HIGH,       requires_approval: true,  credential: write, min_confidence: 0.90 }

  # Destructive — always human approval, always admin credential, no confidence bypass
  drop_database:            { risk: DESTRUCTIVE, requires_approval: true, credential: admin, min_confidence: 999 }
  terminate_instance:       { risk: DESTRUCTIVE, requires_approval: true, credential: admin, min_confidence: 999 }
  delete_s3_bucket:         { risk: DESTRUCTIVE, requires_approval: true, credential: admin, min_confidence: 999 }
  revoke_iam_role:          { risk: DESTRUCTIVE, requires_approval: true, credential: admin, min_confidence: 999 }
```

---

## 5. confidence-thresholds.yaml

```yaml
# policy/confidence-thresholds.yaml

version: "1.0"

# Minimum agent confidence score required to execute without additional friction
thresholds:
  READ_ONLY:    0.00    # no threshold — read anything
  LOW:          0.70    # minor writes — need reasonable confidence
  MEDIUM:       0.80    # significant writes — need high confidence + human
  HIGH:         0.90    # risky writes — need very high confidence + human
  DESTRUCTIVE:  999.0   # impossible threshold — always human, no bypass

# Behavior when confidence is below threshold
below_threshold_action:
  READ_ONLY:    ALLOW           # always allow reads
  LOW:          INJECT_WARNING  # inject warning into context, allow but log
  MEDIUM:       REQUIRE_APPROVAL
  HIGH:         REQUIRE_APPROVAL
  DESTRUCTIVE:  REQUIRE_APPROVAL

# Skillless run overrides
# When was_skillless=true, raise all thresholds by this delta
skillless_threshold_delta: 0.10
# e.g. LOW threshold becomes 0.80, MEDIUM becomes 0.90

# Novel situation overrides
# When situation_class=NOVEL, all writes require approval
novel_situation_override:
  all_writes: REQUIRE_APPROVAL
  disable_confidence_bypass: true
```

---

## 6. budget-rules.yaml

```yaml
# policy/budget-rules.yaml

version: "1.0"

# Default budgets by ticket priority
# Tenants can override these in their tenant YAML
defaults:
  P1:
    max_cost_usd: 2.00
    warn_at_pct: 80          # warn when cost > $1.60
    pause_at_pct: 100        # pause and require human when cost hits $2.00
    max_steps: 30
    max_duration_mins: 60
    max_model_calls: 40
    low_confidence_budget_multiplier: 1.5   # uncertain runs get 50% more budget

  P2:
    max_cost_usd: 1.00
    warn_at_pct: 80
    pause_at_pct: 100
    max_steps: 20
    max_duration_mins: 120
    max_model_calls: 25
    low_confidence_budget_multiplier: 1.3

  P3:
    max_cost_usd: 0.50
    warn_at_pct: 75
    pause_at_pct: 100
    max_steps: 15
    max_duration_mins: 480
    max_model_calls: 15
    low_confidence_budget_multiplier: 1.2

  P4:
    max_cost_usd: 0.25
    warn_at_pct: 70
    pause_at_pct: 100
    max_steps: 10
    max_duration_mins: 1440
    max_model_calls: 10
    low_confidence_budget_multiplier: 1.1

# What happens when budget is hit
budget_exceeded_action:
  on_warn:    LOG_AND_NOTIFY_ONCALL     # slack message to oncall, run continues
  on_pause:   SUSPEND_AND_REQUEST_APPROVAL
  on_absolute_max: FORCE_STOP           # hard stop, escalate

# Per-agent budget sub-limits (prevent one agent from consuming everything)
agent_budget_limits:
  TRIAGE:           { max_cost_usd: 0.05, max_model_calls: 2 }
  DIAGNOSIS:        { max_cost_usd: 0.40, max_model_calls: 15 }
  RESOLUTION:       { max_cost_usd: 0.60, max_model_calls: 20 }
  VERIFICATION:     { max_cost_usd: 0.10, max_model_calls: 4 }
  COMMUNICATION:    { max_cost_usd: 0.05, max_model_calls: 3 }
  POST_INCIDENT:    { max_cost_usd: 0.08, max_model_calls: 4 }
  KNOWLEDGE_CREATOR: { max_cost_usd: 0.15, max_model_calls: 8 }
```

---

## 7. rate-limits.yaml

```yaml
# policy/rate-limits.yaml

version: "1.0"

# Max calls to each tool per run (prevent runaway loops)
per_run:
  get_server_logs:        20
  run_command:            30
  execute_runbook_step:   10
  notify_user:            10
  request_human_approval: 5
  search_knowledge_base:  15
  search_past_incidents:  10
  update_ticket:          5
  escalate:               3
  write_to_filesystem:    100   # high limit — agents write lots of scratchpads

# Max calls per tool per minute across all runs (system-wide, prevents abuse)
per_minute_global:
  run_command:            200
  get_server_logs:        100
  execute_runbook_step:   50
  notify_user:            100
  search_knowledge_base:  300
  update_ticket:          100

# Max concurrent runs per tenant
concurrent_runs:
  default:  10
  P1:       5     # P1s get dedicated capacity

# What happens on rate limit hit
rate_limit_exceeded_action: BLOCK_WITH_BACKOFF
backoff_seconds: 30
```

---

## 8. forbidden-patterns.yaml

```yaml
# policy/forbidden-patterns.yaml

version: "1.0"

# These patterns are checked against the FULL command string after template substitution.
# Match = BLOCK. No exceptions. No confidence bypass. No approval path.

shell_commands:
  - pattern: 'rm\s+-rf'
    reason: "Recursive delete"
  - pattern: 'dd\s+if='
    reason: "Disk write operation"
  - pattern: '>\s*/dev/sd'
    reason: "Direct disk write"
  - pattern: 'chmod\s+777'
    reason: "World-writable permissions"
  - pattern: 'chown\s+root'
    reason: "Privilege escalation"
  - pattern: 'sudo\s+su'
    reason: "Root shell"
  - pattern: 'passwd\s'
    reason: "Password change"
  - pattern: 'crontab\s+-'
    reason: "Cron modification"
  - pattern: 'iptables\s+-F'
    reason: "Firewall flush"
  - pattern: 'ufw\s+disable'
    reason: "Firewall disable"

database:
  - pattern: 'DROP\s+DATABASE'
    reason: "Database drop"
  - pattern: 'DROP\s+TABLE'
    reason: "Table drop"
  - pattern: 'TRUNCATE\s+TABLE'
    reason: "Table truncation"
  - pattern: 'DELETE\s+FROM\s+\w+\s*;'
    reason: "Unfiltered delete (no WHERE clause)"
  - pattern: 'ALTER\s+TABLE.*DROP\s+COLUMN'
    reason: "Column drop"
  - pattern: 'DROP\s+SCHEMA'
    reason: "Schema drop"

network:
  - pattern: 'curl.*\|\s*(bash|sh)'
    reason: "Remote code execution via pipe"
  - pattern: 'wget.*\|\s*(bash|sh)'
    reason: "Remote code execution via pipe"
  - pattern: 'nc\s+-e'
    reason: "Netcat reverse shell"
  - pattern: 'python.*-c.*socket'
    reason: "Python reverse shell pattern"

kubernetes:
  - pattern: 'kubectl\s+delete\s+namespace'
    reason: "Namespace deletion"
  - pattern: 'kubectl\s+delete\s+pv'
    reason: "Persistent volume deletion"
  - pattern: 'kubectl.*--all-namespaces.*delete'
    reason: "Cross-namespace delete"

aws:
  - pattern: 'aws\s+s3\s+rm\s+.*--recursive'
    reason: "Recursive S3 delete"
  - pattern: 'aws\s+ec2\s+terminate-instances'
    reason: "Instance termination (use command_id instead)"
  - pattern: 'aws\s+iam\s+delete'
    reason: "IAM deletion"
  - pattern: 'aws\s+rds\s+delete'
    reason: "RDS deletion"
```

---

## 9. destructive-actions.yaml

```yaml
# policy/destructive-actions.yaml

version: "1.0"

# Actions that ALWAYS require human approval.
# Confidence score cannot bypass these.
# Even a confidence of 0.99 still requires human sign-off.

always_require_approval:
  - id: any_destructive_command
    match: { risk_level: DESTRUCTIVE }
    reason: "Destructive actions always require human approval"
    notification:
      channel: slack
      mention: "@oncall-lead"
      timeout_mins: 10

  - id: production_database_write
    match:
      tool: run_command
      environment: production
      command_category: database_write
    reason: "Production database writes require human approval"
    notification:
      channel: slack
      timeout_mins: 15

  - id: scale_down
    match:
      tool: run_command
      command_id: scale_deployment
      params:
        direction: down
    reason: "Scaling down in production requires human approval"
    notification:
      channel: slack
      timeout_mins: 10

  - id: novel_situation_writes
    match:
      situation_class: NOVEL
      tool_risk: [LOW, MEDIUM, HIGH, DESTRUCTIVE]
    reason: "Novel situation — no historical precedent. Human oversight required for all writes."
    notification:
      channel: slack
      timeout_mins: 20

  - id: low_confidence_medium_risk
    match:
      risk_level: MEDIUM
      agent_confidence_below: 0.70
    reason: "Confidence below safe threshold for medium-risk operation"
    notification:
      channel: slack
      timeout_mins: 10

  - id: degradation_detected
    match:
      situation_class: DEGRADATION_DETECTED
    reason: "Last action worsened system state. All writes frozen until human reviews."
    notification:
      channel: slack
      mention: "@oncall-lead @team-lead"
      timeout_mins: 5
      urgency: HIGH
```

---

## 10. Tenant Override Example

```yaml
# policy/tenants/{tenant_id}.yaml
# Overrides for a specific tenant. Only listed keys are overridden.

version: "1.0"
tenant_id: "acme-corp"
tenant_name: "Acme Corporation"

overrides:
  budgets:
    P1:
      max_cost_usd: 5.00        # Acme has higher budget for P1
      max_steps: 40
    P3:
      max_cost_usd: 0.10        # Acme wants tighter P3 budget

  confidence_thresholds:
    MEDIUM: 0.85                # Acme is more conservative — higher confidence required

  approval_timeout_mins: 10     # Acme's oncall responds fast

  # Additional forbidden patterns for this tenant
  additional_forbidden_patterns:
    - pattern: 'mysql\s+-u\s+root'
      reason: "Acme policy: no root MySQL access from agents"

  # Tenant-specific rate limits
  rate_limits:
    per_run:
      notify_user: 5            # Acme wants fewer notifications

  # Access method restrictions
  allowed_access_methods:
    - SSM                       # Acme only allows SSM, no SSH bastion
    - CLOUDWATCH_API
    - K8S_EXEC

  # Tools completely disabled for this tenant
  disabled_tools:
    - escalate                  # Acme handles escalation through their own system
```

---

## 11. PolicyGuard Evaluation Order

PolicyGuard evaluates gates in this exact order.
First gate to fire wins. Later gates do not run if an earlier one blocked.

```
Gate 1 — Agent stopped or paused
  Check: is run.stop_signal or run.paused set in Redis?
  If yes: BLOCK (run is suspended)

Gate 2 — Authority check
  Check: is this tool in this agent_role's allowed_tools list?
  Source: authority-matrix.yaml
  If no: BLOCK, reason="Agent role {role} is not permitted to call {tool}"

Gate 3 — Global forbidden patterns
  Check: does the resolved command match any pattern in forbidden-patterns.yaml?
  Source: forbidden-patterns.yaml + tenant additional_forbidden_patterns
  If yes: BLOCK, reason="Forbidden pattern: {pattern} — {reason}"

Gate 4 — Shell injection check
  Check: does any parameter value contain shell metacharacters?
  Source: base.yaml always_block rules
  If yes: BLOCK, reason="Shell injection pattern detected"

Gate 5 — Credential pattern check
  Check: does the input contain anything that looks like a credential?
  Source: base.yaml always_block rules
  If yes: BLOCK, reason="Credential-like string in input"

Gate 6 — Scope check
  Check: does the input reference resources belonging to this tenant only?
  Source: run context
  If no: BLOCK, reason="Cross-tenant resource reference"

Gate 7 — Rate limit check
  Check: has this tool been called more than rate_limit times this run?
  Source: rate-limits.yaml, Redis counter
  If yes: BLOCK, reason="Rate limit: {tool} called {n} times (max {max})"

Gate 8 — Circuit breaker check
  Check: is the circuit for this tool OPEN?
  Source: circuit_breakers table / Redis
  If open: BLOCK, reason="Circuit open for {tool} — failure rate {rate}%"

Gate 9 — Budget check
  Check: is cost_so_far >= max_cost_usd?
  Source: budget-rules.yaml, agent_runs.cost_usd
  If yes: BLOCK, reason="Budget ceiling reached — human approval required"

Gate 10 — Risk level resolution
  Determine: what is the risk level of this tool/command?
  Source: risk-levels.yaml
  Output: risk_level (READ_ONLY | LOW | MEDIUM | HIGH | DESTRUCTIVE)

Gate 11 — Destructive action check
  Check: does this action match any rule in destructive-actions.yaml?
  If yes: REQUIRE_APPROVAL (always, regardless of confidence)

Gate 12 — Novel/degradation situation override
  Check: is situation_class NOVEL or DEGRADATION_DETECTED?
  If DEGRADATION_DETECTED: BLOCK all writes until human reviews
  If NOVEL + write tool: REQUIRE_APPROVAL

Gate 13 — Confidence threshold check
  Check: is agent_confidence >= threshold for this risk_level?
  Source: confidence-thresholds.yaml
  If below threshold:
    READ_ONLY → ALLOW
    LOW       → ALLOW with warning logged
    MEDIUM    → REQUIRE_APPROVAL
    HIGH      → REQUIRE_APPROVAL
    DESTRUCTIVE → REQUIRE_APPROVAL (already caught at Gate 11)

Gate 14 — Requires approval flag
  Check: does risk-levels.yaml mark this tool/command as requires_approval: true?
  If yes: REQUIRE_APPROVAL

Gate 15 — Input schema validation
  Check: does the input match the tool's JSON schema exactly?
  If no: BLOCK, reason="Input schema violation: {errors}"

Gate 16 — Parameter source validation
  Check: for run_command, are all parameters sourced from service_metadata
         (not from LLM-generated values in restricted fields)?
  If violation: BLOCK, reason="Parameter must come from service_metadata"

→ If all gates pass: ALLOW
```

---

## 12. PolicyGuard Response Schema

```typescript
interface PolicyDecision {
  allowed: boolean;
  decision: 'ALLOWED' | 'BLOCKED' | 'APPROVAL_REQUIRED';
  gate_fired: string | null;        // which gate made the decision
  reason: string;
  risk_level: RiskLevel;
  required_approvals: string[];     // who must approve (role or person IDs)
  approval_timeout_mins: number;
  modified_input: object | null;    // sanitized input (strips disallowed fields)
  warnings: string[];               // non-blocking warnings to log
  audit_entry: {
    tool_name: string;
    input_hash: string;
    agent_role: string;
    confidence: number;
    cost_at_time: number;
  };
}
```

---

## 13. Loading and Hot-Reloading Policies

```typescript
// PolicyGuard loads policies on startup.
// Database overrides are merged on top of file policies.
// Hot reload happens every 60 seconds without restart.

class PolicyLoader {
  async load(): Promise<MergedPolicy> {
    const base    = await loadYaml('policy/base.yaml');
    const matrix  = await loadYaml('policy/authority-matrix.yaml');
    const risk    = await loadYaml('policy/risk-levels.yaml');
    const conf    = await loadYaml('policy/confidence-thresholds.yaml');
    const budget  = await loadYaml('policy/budget-rules.yaml');
    const limits  = await loadYaml('policy/rate-limits.yaml');
    const forb    = await loadYaml('policy/forbidden-patterns.yaml');
    const destr   = await loadYaml('policy/destructive-actions.yaml');

    // Tenant-specific overrides from DB
    const dbOverrides = await db.query(
      'SELECT config FROM policies WHERE active = true AND (tenant_id = $1 OR tenant_id IS NULL)',
      [tenantId]
    );

    return merge(base, matrix, risk, conf, budget, limits, forb, destr, ...dbOverrides);
  }
}

// Any policy change in the database takes effect within 60 seconds.
// No deploy required for: budget changes, threshold changes, new forbidden patterns.
// Requires deploy for: new gate logic, new tool definitions, new agent roles.
```
