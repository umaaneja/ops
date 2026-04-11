# Data Models — Ticket Agent System
# docs/13-data-models.md
# Drop this file into your docs/ folder. Claude Code reads it as part of the architecture spec.

---

## Overview

Five storage backends. Each has a distinct responsibility.
Never consolidate them — they serve different timescales and access patterns.

```
PostgreSQL      → durable relational state, episodic memory, audit log
Redis           → working memory per run, TTL-scoped, in-flight state
Neo4j           → knowledge graph, entity relationships, failure patterns
Elasticsearch   → semantic knowledge base, hybrid BM25 + vector search
S3 / Filesystem → workspace files per run, logs, scratchpads, drafts
```

---

## 1. PostgreSQL — Full Schema

Run migrations in this exact order.
Each migration file is named and must be idempotent (use IF NOT EXISTS everywhere).

---

### Migration 001 — Core extensions

```sql
-- 001_extensions.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "vector";          -- pgvector for embeddings
CREATE EXTENSION IF NOT EXISTS "pg_trgm";         -- trigram search on text fields
CREATE EXTENSION IF NOT EXISTS "btree_gin";       -- GIN indexes on composite types
```

---

### Migration 002 — Enumerations

```sql
-- 002_enums.sql

CREATE TYPE run_state AS ENUM (
  'IDLE',
  'PLANNING',
  'DISPATCHING',
  'EXECUTING',
  'REPLANNING',
  'COMPLETING',
  'DONE',
  'FAILED',
  'INTERRUPTED',
  'HUMAN_TAKEOVER'
);

CREATE TYPE step_status AS ENUM (
  'PENDING',
  'RUNNING',
  'SUCCESS',
  'FAILED',
  'SKIPPED',
  'BLOCKED',
  'AWAITING_APPROVAL'
);

CREATE TYPE outcome AS ENUM (
  'RESOLVED',
  'ESCALATED',
  'FAILED',
  'INTERRUPTED',
  'HUMAN_TAKEOVER'
);

CREATE TYPE approval_status AS ENUM (
  'PENDING',
  'APPROVED',
  'REJECTED',
  'TIMEOUT',
  'CANCELLED'
);

CREATE TYPE risk_level AS ENUM (
  'READ_ONLY',
  'LOW',
  'MEDIUM',
  'HIGH',
  'DESTRUCTIVE'
);

CREATE TYPE agent_role AS ENUM (
  'TRIAGE',
  'DIAGNOSIS',
  'RESOLUTION',
  'VERIFICATION',
  'COMMUNICATION',
  'POST_INCIDENT',
  'KNOWLEDGE_CREATOR'
);

CREATE TYPE access_method AS ENUM (
  'SSM',
  'SSH_BASTION',
  'KUBECTL_EXEC',
  'API',
  'AGENT_RUNNER',
  'CLOUDWATCH_API',
  'S3_API'
);

CREATE TYPE auth_method AS ENUM (
  'IAM_ROLE',
  'VAULT_PATH',
  'SSM_PARAM',
  'K8S_SA',
  'OAUTH_TOKEN'
);

CREATE TYPE host_resolver AS ENUM (
  'STATIC',
  'AWS_ASG',
  'K8S_POD',
  'CONSUL',
  'DNS'
);

CREATE TYPE gap_type AS ENUM (
  'MISSING_SKILL',
  'INCOMPLETE_SKILL',
  'WRONG_GUIDANCE',
  'MISSING_DOCUMENT',
  'STALE_DOCUMENT',
  'WRONG_CHUNK_BOUNDARY',
  'LOW_CONFIDENCE'
);

CREATE TYPE review_status AS ENUM (
  'PENDING_REVIEW',
  'APPROVED',
  'REJECTED',
  'PUBLISHED'
);
```

---

### Migration 003 — Tenants and teams

```sql
-- 003_tenants.sql

CREATE TABLE tenants (
  tenant_id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name                TEXT NOT NULL UNIQUE,
  config              JSONB NOT NULL DEFAULT '{}',
    -- max_cost_usd_p1: decimal
    -- max_cost_usd_p2: decimal
    -- max_steps_default: int
    -- approval_timeout_mins: int
    -- slack_webhook_url: text
    -- allowed_access_methods: text[]
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE teams (
  team_id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
  name                TEXT NOT NULL,
  slack_channel       TEXT,
  oncall_schedule_id  TEXT,                         -- PagerDuty / OpsGenie schedule ID
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE persons (
  person_id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
  team_id             UUID REFERENCES teams(team_id),
  name                TEXT NOT NULL,
  email               TEXT NOT NULL,
  role                TEXT,
  slack_user_id       TEXT,
  is_oncall           BOOLEAN NOT NULL DEFAULT FALSE,
  oncall_since        TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### Migration 004 — Agent runs (core orchestrator table)

```sql
-- 004_agent_runs.sql

CREATE TABLE agent_runs (
  -- Identity
  run_id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id             UUID NOT NULL REFERENCES tenants(tenant_id),
  ticket_id             TEXT NOT NULL,               -- external ticket ID (Jira, ServiceNow, etc.)
  ticket_source         TEXT NOT NULL,               -- 'jira' | 'servicenow' | 'zendesk' | 'pagerduty'

  -- State (only orchestrator writes this column)
  state                 run_state NOT NULL DEFAULT 'IDLE',
  previous_state        run_state,                   -- for audit trail of transitions
  state_changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  state_change_reason   TEXT,

  -- Plan (versioned, never mutated)
  plan_version          INT NOT NULL DEFAULT 0,      -- increments on each replan
  plan_json             JSONB,                       -- current active plan
  plan_history          JSONB NOT NULL DEFAULT '[]', -- array of all prior plan versions

  -- Execution
  step_index            INT NOT NULL DEFAULT 0,
  total_steps           INT,
  active_agent          agent_role,
  active_agent_id       UUID,                        -- FK to agent_instances

  -- Cost and budget
  cost_usd              DECIMAL(10,6) NOT NULL DEFAULT 0,
  max_cost_usd          DECIMAL(10,4) NOT NULL DEFAULT 0.50,
  cost_warned           BOOLEAN NOT NULL DEFAULT FALSE,  -- true when cost > 80% of max

  -- Timing
  started_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deadline_at           TIMESTAMPTZ,
  completed_at          TIMESTAMPTZ,

  -- Outcome
  outcome               outcome,
  outcome_reason        TEXT,
  resolution_summary    TEXT,

  -- Skill tracking
  skill_set             TEXT[] NOT NULL DEFAULT '{}',   -- skills loaded for this run
  was_skillless         BOOLEAN NOT NULL DEFAULT FALSE,
  skill_gap_detected    BOOLEAN NOT NULL DEFAULT FALSE,

  -- Configuration snapshot (frozen at run start)
  harness_config_version TEXT NOT NULL DEFAULT 'unknown',
  model_config          JSONB NOT NULL DEFAULT '{}',

  -- Metadata
  attempt_number        INT NOT NULL DEFAULT 1,
  parent_run_id         UUID REFERENCES agent_runs(run_id),  -- for parallel sub-runs
  tags                  TEXT[] NOT NULL DEFAULT '{}',
  metadata              JSONB NOT NULL DEFAULT '{}',

  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agent_runs_tenant_state     ON agent_runs(tenant_id, state);
CREATE INDEX idx_agent_runs_ticket_id        ON agent_runs(ticket_id);
CREATE INDEX idx_agent_runs_state_created    ON agent_runs(state, created_at DESC);
CREATE INDEX idx_agent_runs_active           ON agent_runs(tenant_id, state)
  WHERE state NOT IN ('DONE', 'FAILED', 'INTERRUPTED');
```

---

### Migration 005 — Agent steps

```sql
-- 005_agent_steps.sql

CREATE TABLE agent_steps (
  step_id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  run_id              UUID NOT NULL REFERENCES agent_runs(run_id) ON DELETE CASCADE,
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  -- Position
  step_index          INT NOT NULL,
  plan_version        INT NOT NULL,                  -- which plan version produced this step

  -- Agent
  agent_role          agent_role NOT NULL,
  agent_instance_id   UUID,

  -- Tool call
  tool_name           TEXT NOT NULL,
  command_id          TEXT,                          -- if run_command, which template was used
  input_json          JSONB NOT NULL DEFAULT '{}',
  input_hash          TEXT,                          -- SHA256 of input for idempotency check

  -- Output
  output_json         JSONB,
  output_summary      TEXT,                          -- truncated human-readable summary
  exit_code           INT,
  stderr_summary      TEXT,

  -- Status
  status              step_status NOT NULL DEFAULT 'PENDING',
  error_message       TEXT,
  error_type          TEXT,                          -- 'TOOL_ERROR' | 'MODEL_ERROR' | 'POLICY_BLOCK' | 'TIMEOUT'
  retry_count         INT NOT NULL DEFAULT 0,

  -- Verification
  verification_result JSONB,                         -- from Verification Agent
  verification_confidence DECIMAL(4,3),
  verified_at         TIMESTAMPTZ,

  -- Cost
  cost_usd            DECIMAL(10,6) NOT NULL DEFAULT 0,
  token_count_in      INT,
  token_count_out     INT,
  latency_ms          INT,

  -- Timing
  started_at          TIMESTAMPTZ,
  ended_at            TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_steps_run_id         ON agent_steps(run_id, step_index);
CREATE INDEX idx_steps_status         ON agent_steps(status) WHERE status = 'RUNNING';
CREATE INDEX idx_steps_tool_name      ON agent_steps(tool_name, created_at DESC);
```

---

### Migration 006 — Tool calls (idempotency + audit)

```sql
-- 006_tool_calls.sql

CREATE TABLE tool_calls (
  -- Idempotency key: hash(run_id || step_index || tool_name || input)
  call_id             TEXT PRIMARY KEY,

  run_id              UUID NOT NULL REFERENCES agent_runs(run_id) ON DELETE CASCADE,
  step_id             UUID NOT NULL REFERENCES agent_steps(step_id) ON DELETE CASCADE,
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  -- What was called
  tool_name           TEXT NOT NULL,
  command_id          TEXT,
  input_hash          TEXT NOT NULL,
  server_host         TEXT,                          -- sanitized, no internal IP
  access_method       access_method,

  -- Was it a cache hit?
  cache_hit           BOOLEAN NOT NULL DEFAULT FALSE,

  -- Result
  output_json         JSONB,
  success             BOOLEAN,
  exit_code           INT,
  error_message       TEXT,

  -- Performance
  latency_ms          INT,
  bytes_transferred   INT,

  -- Policy decision
  policy_decision     TEXT NOT NULL DEFAULT 'UNKNOWN', -- 'ALLOWED' | 'BLOCKED' | 'APPROVAL_REQUIRED'
  policy_block_reason TEXT,

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tool_calls_run_id     ON tool_calls(run_id);
CREATE INDEX idx_tool_calls_tool_name  ON tool_calls(tool_name, created_at DESC);
CREATE INDEX idx_tool_calls_cache      ON tool_calls(cache_hit, tool_name) WHERE cache_hit = TRUE;
```

---

### Migration 007 — Human approvals

```sql
-- 007_approvals.sql

CREATE TABLE approvals (
  approval_id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  run_id              UUID NOT NULL REFERENCES agent_runs(run_id) ON DELETE CASCADE,
  step_id             UUID NOT NULL REFERENCES agent_steps(step_id) ON DELETE CASCADE,
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  -- What needs approval
  action_description  TEXT NOT NULL,
  tool_name           TEXT NOT NULL,
  command_id          TEXT,
  payload_json        JSONB NOT NULL,                -- full action payload for display
  risk_level          risk_level NOT NULL,
  agent_confidence    DECIMAL(4,3),
  confidence_threshold DECIMAL(4,3),
  rollback_available  BOOLEAN NOT NULL DEFAULT FALSE,
  rollback_description TEXT,

  -- Decision
  status              approval_status NOT NULL DEFAULT 'PENDING',
  decided_by          TEXT,                          -- user ID or email
  decided_at          TIMESTAMPTZ,
  decision_reason     TEXT,

  -- Notification
  notification_sent_at TIMESTAMPTZ,
  notification_channel TEXT,                         -- 'slack' | 'email' | 'webhook'
  notification_ref    TEXT,                          -- Slack message ID etc.

  -- Timeout
  timeout_at          TIMESTAMPTZ NOT NULL,          -- auto-escalate after this

  -- Immutable audit
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Approvals are append-only. No updates to core fields after creation.
-- Only status, decided_by, decided_at, decision_reason can be updated.
CREATE INDEX idx_approvals_run_id      ON approvals(run_id);
CREATE INDEX idx_approvals_pending     ON approvals(status, timeout_at)
  WHERE status = 'PENDING';
CREATE INDEX idx_approvals_decided_by  ON approvals(decided_by, decided_at DESC)
  WHERE decided_by IS NOT NULL;
```

---

### Migration 008 — Policy audit log (immutable)

```sql
-- 008_policy_audit.sql

-- This table is append-only. No rows are ever updated or deleted.
-- It is the immutable record of every policy decision made by the system.
CREATE TABLE policy_audit (
  audit_id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  run_id              UUID NOT NULL,                 -- no FK — audit log survives run deletion
  step_id             UUID,
  tenant_id           UUID NOT NULL,
  agent_role          agent_role NOT NULL,

  -- What was evaluated
  tool_name           TEXT NOT NULL,
  input_hash          TEXT,
  risk_level          risk_level,
  agent_confidence    DECIMAL(4,3),

  -- Policy decision
  decision            TEXT NOT NULL,                 -- 'ALLOWED' | 'BLOCKED' | 'APPROVAL_REQUIRED'
  gate_that_fired     TEXT,                          -- which gate made the decision
  reason              TEXT NOT NULL,

  -- Context
  cost_at_time        DECIMAL(10,6),
  budget_at_time      DECIMAL(10,4),
  circuit_state       TEXT,                          -- 'CLOSED' | 'OPEN' | 'HALF_OPEN'

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_policy_audit_run_id   ON policy_audit(run_id, created_at);
CREATE INDEX idx_policy_audit_decision ON policy_audit(decision, created_at DESC);
CREATE INDEX idx_policy_audit_tool     ON policy_audit(tool_name, decision, created_at DESC);
```

---

### Migration 009 — Episodic memory

```sql
-- 009_episodes.sql

CREATE TABLE episodes (
  episode_id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  -- Ticket reference
  ticket_id           TEXT NOT NULL,
  run_id              UUID NOT NULL,                 -- no FK — episodes survive run cleanup
  external_url        TEXT,                          -- link to ticket in source system

  -- Classification
  category            TEXT,
  affected_service    TEXT,
  priority            TEXT,

  -- Embeddings (for similarity search)
  ticket_embedding    VECTOR(1536),                  -- embedding of ticket summary
  resolution_embedding VECTOR(1536),                 -- embedding of resolution text

  -- Summary content
  ticket_summary      TEXT NOT NULL,
  root_cause          TEXT,
  root_cause_category TEXT,
  resolution_text     TEXT,
  resolution_action   TEXT,                          -- main action that fixed it

  -- Outcome
  outcome             outcome NOT NULL,
  steps_taken         INT,
  replan_count        INT NOT NULL DEFAULT 0,
  total_cost_usd      DECIMAL(10,6),
  total_duration_ms   BIGINT,

  -- Skill information
  skill_set_used      TEXT[] NOT NULL DEFAULT '{}',
  was_skillless       BOOLEAN NOT NULL DEFAULT FALSE,
  what_skill_would_have_helped TEXT,

  -- Enrichment from Knowledge Creator
  hypothesis_trace    JSONB NOT NULL DEFAULT '[]',
    -- [{hypothesis, introduced_at_step, confidence_at_introduction,
    --   confidence_at_peak, was_correct, evidence_that_confirmed_it,
    --   steps_to_confirm, dead_end, why_abandoned}]

  diagnostic_usefulness JSONB NOT NULL DEFAULT '[]',
    -- [{tool, query, was_useful, step_at_which_useful, what_it_revealed}]

  reasoning_quality   JSONB,
    -- {first_hypothesis_correct, correct_hypothesis_introduced_at_step,
    --   steps_wasted_on_wrong_hypotheses, diagnostic_order_efficiency,
    --   optimal_diagnostic_sequence, confidence_calibration_score}

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Vector similarity indexes (IVFFlat for approximate search)
CREATE INDEX idx_episodes_ticket_embedding    ON episodes USING ivfflat (ticket_embedding vector_cosine_ops)
  WITH (lists = 100);
CREATE INDEX idx_episodes_resolution_embedding ON episodes USING ivfflat (resolution_embedding vector_cosine_ops)
  WITH (lists = 100);

CREATE INDEX idx_episodes_tenant_category     ON episodes(tenant_id, category, created_at DESC);
CREATE INDEX idx_episodes_tenant_outcome      ON episodes(tenant_id, outcome, created_at DESC);
CREATE INDEX idx_episodes_affected_service    ON episodes(tenant_id, affected_service, created_at DESC);
CREATE INDEX idx_episodes_skillless           ON episodes(tenant_id, was_skillless)
  WHERE was_skillless = TRUE;
```

---

### Migration 010 — Episode steps (detailed step history per episode)

```sql
-- 010_episode_steps.sql

CREATE TABLE episode_steps (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  episode_id          UUID NOT NULL REFERENCES episodes(episode_id) ON DELETE CASCADE,
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  step_index          INT NOT NULL,
  agent_role          agent_role NOT NULL,
  tool_name           TEXT NOT NULL,
  command_id          TEXT,

  input_summary       TEXT,                          -- truncated, not full input
  outcome             step_status NOT NULL,
  error_summary       TEXT,

  was_useful          BOOLEAN,                       -- updated by nightly job
  confidence_before   DECIMAL(4,3),
  confidence_after    DECIMAL(4,3),
  confidence_delta    DECIMAL(4,3),

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ep_steps_episode_id  ON episode_steps(episode_id, step_index);
CREATE INDEX idx_ep_steps_useful      ON episode_steps(episode_id, was_useful)
  WHERE was_useful = TRUE;
CREATE INDEX idx_ep_steps_tool        ON episode_steps(tool_name, was_useful, created_at DESC);
```

---

### Migration 011 — Knowledge gaps and drafts

```sql
-- 011_knowledge_gaps.sql

CREATE TABLE knowledge_gap_reports (
  gap_id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
  run_id              UUID NOT NULL,
  episode_id          UUID REFERENCES episodes(episode_id),

  gap_type            gap_type NOT NULL,
  priority            TEXT NOT NULL DEFAULT 'MEDIUM',  -- 'HIGH' | 'MEDIUM' | 'LOW'

  -- What is missing
  description         TEXT NOT NULL,
  affected_service    TEXT,
  category            TEXT,
  estimated_steps_saved INT,
  estimated_cost_saved_usd DECIMAL(8,4),

  -- Evidence
  kb_queries_that_failed  TEXT[],
  what_actually_worked    TEXT,

  -- Suggestion
  suggested_skill_name    TEXT,
  suggested_document_title TEXT,
  suggested_content_outline JSONB,

  -- Review
  status              review_status NOT NULL DEFAULT 'PENDING_REVIEW',
  reviewed_by         TEXT,
  reviewed_at         TIMESTAMPTZ,
  review_notes        TEXT,

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- KB document drafts (generated by Knowledge Creator, reviewed by humans)
CREATE TABLE kb_drafts (
  draft_id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
  gap_id              UUID REFERENCES knowledge_gap_reports(gap_id),
  run_id              UUID NOT NULL,

  draft_type          TEXT NOT NULL,                  -- 'RUNBOOK' | 'SOP' | 'KNOWN_ISSUE' | 'FAQ'
  title               TEXT NOT NULL,
  content_markdown    TEXT NOT NULL,                  -- full draft content
  affected_service    TEXT,
  category            TEXT,

  status              review_status NOT NULL DEFAULT 'PENDING_REVIEW',
  reviewed_by         TEXT,
  reviewed_at         TIMESTAMPTZ,
  published_at        TIMESTAMPTZ,
  published_chunk_ids TEXT[],                         -- IDs of chunks in KB after publish

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Skill drafts
CREATE TABLE skill_drafts (
  draft_id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
  gap_id              UUID REFERENCES knowledge_gap_reports(gap_id),
  run_id              UUID NOT NULL,

  skill_name          TEXT NOT NULL,
  skill_version       TEXT NOT NULL DEFAULT '0.1.0',
  content_markdown    TEXT NOT NULL,                  -- full SKILL.md content
  affected_categories TEXT[],
  affected_services   TEXT[],
  estimated_steps_saved INT,

  status              review_status NOT NULL DEFAULT 'PENDING_REVIEW',
  reviewed_by         TEXT,
  reviewed_at         TIMESTAMPTZ,
  published_at        TIMESTAMPTZ,

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### Migration 012 — Skills usage tracking

```sql
-- 012_skills_usage.sql

CREATE TABLE skills_usage (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  run_id              UUID NOT NULL,
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
  episode_id          UUID REFERENCES episodes(episode_id),

  skill_name          TEXT NOT NULL,
  skill_version       TEXT NOT NULL,
  loaded_at           TIMESTAMPTZ NOT NULL,
  unloaded_at         TIMESTAMPTZ,

  -- Did the agent actually follow this skill's guidance?
  was_used            BOOLEAN,                        -- set by post-incident analysis
  aligned_tool_calls  INT,                            -- tool calls that matched skill guidance
  total_tool_calls    INT,
  alignment_ratio     DECIMAL(4,3),

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_skills_usage_skill_name  ON skills_usage(skill_name, created_at DESC);
CREATE INDEX idx_skills_usage_run_id      ON skills_usage(run_id);
```

---

### Migration 013 — Server access registry

```sql
-- 013_server_access_registry.sql

CREATE TABLE service_access_registry (
  registry_id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  -- Service identity
  service_name        TEXT NOT NULL,
  environment         TEXT NOT NULL,                  -- 'production' | 'staging' | 'canary'
  team_id             UUID REFERENCES teams(team_id),

  -- Connection
  access_method       access_method NOT NULL,
  host_resolver       host_resolver NOT NULL,
  host_pattern        TEXT,                           -- static host, ASG tag, k8s selector, or DNS pattern
  bastion_host        TEXT,
  port                INT DEFAULT 22,
  k8s_namespace       TEXT,

  -- Auth (references only — no credentials stored here)
  auth_method         auth_method NOT NULL,
  auth_reference      TEXT NOT NULL,                  -- ARN, vault path, SSM param name, k8s SA name
  read_scope          TEXT NOT NULL DEFAULT 'read_only',
  write_scope         TEXT,

  -- Log locations (array of log source definitions)
  log_locations       JSONB NOT NULL DEFAULT '[]',
    -- [{name, type, path, format, field_map, log_group, log_stream_pattern, region}]

  -- Allowed commands (parameterized templates)
  allowed_commands    JSONB NOT NULL DEFAULT '[]',
    -- [{id, template, params, param_source, param_constraints,
    --   read_only, requires_approval_if_confidence_below, risk_level, description}]

  -- Forbidden patterns (regex)
  forbidden_patterns  TEXT[] NOT NULL DEFAULT '{}',

  -- Approval requirements
  requires_approval   JSONB NOT NULL DEFAULT '[]',
    -- [{pattern, risk_level, reason, confidence_threshold}]

  -- Service metadata (available to agent via templates)
  service_metadata    JSONB NOT NULL DEFAULT '{}',
    -- pg_host, pg_user, db_name, pgbouncer_socket, etc.

  -- Primary selection strategy for multi-host services
  primary_selection_strategy TEXT NOT NULL DEFAULT 'FIRST_HEALTHY',
    -- 'FIRST_HEALTHY' | 'TAG_BASED' | 'CONSUL_PRIMARY' | 'K8S_READY'

  active              BOOLEAN NOT NULL DEFAULT TRUE,
  last_validated_at   TIMESTAMPTZ,
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_by          TEXT,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_sar_service_env_tenant
  ON service_access_registry(tenant_id, service_name, environment)
  WHERE active = TRUE;
```

---

### Migration 014 — Harness configuration versions

```sql
-- 014_harness_configs.sql

CREATE TABLE harness_configs (
  config_id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  version             TEXT NOT NULL,                  -- semver e.g. '1.4.2'
  skill_registry      JSONB NOT NULL,                 -- snapshot of skills/registry.json
  prompt_templates    JSONB NOT NULL,                 -- snapshot of all prompt templates
  tool_manifest_version TEXT NOT NULL,
  model_routing       JSONB NOT NULL,                 -- which model per priority

  deployed_at         TIMESTAMPTZ,
  deployed_by         TEXT,
  is_active           BOOLEAN NOT NULL DEFAULT FALSE,
  shadow_mode         BOOLEAN NOT NULL DEFAULT FALSE,

  -- Eval results before deployment
  eval_pass_rate      DECIMAL(5,4),
  eval_cost_per_ticket DECIMAL(10,6),
  eval_latency_p50_ms INT,
  eval_latency_p99_ms INT,
  eval_run_at         TIMESTAMPTZ,

  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_harness_configs_active
  ON harness_configs(tenant_id, is_active)
  WHERE is_active = TRUE AND shadow_mode = FALSE;
```

---

### Migration 015 — Agent instances

```sql
-- 015_agent_instances.sql

CREATE TABLE agent_instances (
  agent_id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  run_id              UUID NOT NULL REFERENCES agent_runs(run_id) ON DELETE CASCADE,
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),

  role                agent_role NOT NULL,
  model_used          TEXT NOT NULL,
  model_tier          TEXT NOT NULL,                  -- 'fast' | 'capable' | 'powerful'

  -- Skills active for this instance
  skills_loaded       TEXT[] NOT NULL DEFAULT '{}',
  tool_manifest_subset TEXT[] NOT NULL DEFAULT '{}', -- tools this agent can access

  -- Execution
  state               TEXT NOT NULL DEFAULT 'RUNNING', -- 'RUNNING' | 'DONE' | 'FAILED'
  loops_completed     INT NOT NULL DEFAULT 0,
  ralph_loop_count    INT NOT NULL DEFAULT 0,

  -- Cost
  cost_usd            DECIMAL(10,6) NOT NULL DEFAULT 0,
  token_count_in      INT NOT NULL DEFAULT 0,
  token_count_out     INT NOT NULL DEFAULT 0,

  spawned_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at        TIMESTAMPTZ,

  -- Parallel execution tracking
  parent_agent_id     UUID REFERENCES agent_instances(agent_id),
  workspace_subdir    TEXT                            -- e.g. 'diagnostics/agent_a/'
);

CREATE INDEX idx_agent_instances_run_id  ON agent_instances(run_id, role);
```

---

### Migration 016 — Circuit breakers

```sql
-- 016_circuit_breakers.sql

CREATE TABLE circuit_breakers (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
  tool_name           TEXT NOT NULL,

  state               TEXT NOT NULL DEFAULT 'CLOSED',  -- 'CLOSED' | 'OPEN' | 'HALF_OPEN'
  failure_count       INT NOT NULL DEFAULT 0,
  success_count       INT NOT NULL DEFAULT 0,
  last_failure_at     TIMESTAMPTZ,
  opened_at           TIMESTAMPTZ,
  half_open_at        TIMESTAMPTZ,

  -- Rolling window stats (last 60 seconds)
  window_start        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  window_failures     INT NOT NULL DEFAULT 0,
  window_total        INT NOT NULL DEFAULT 0,
  failure_rate        DECIMAL(5,4) NOT NULL DEFAULT 0,

  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_cb_tenant_tool ON circuit_breakers(tenant_id, tool_name);
```

---

## 2. Redis — Key Patterns and TTL

All keys are prefixed with the tenant ID to ensure isolation.

```
Prefix convention:  {tenant_id}:{key_pattern}
Example:            t_abc123:run:run-uuid-here:meta
```

### Working memory keys (per run)

```
KEY                                         TYPE    TTL           DESCRIPTION
───────────────────────────────────────────────────────────────────────────────────────────

run:{run_id}:meta                           HASH    run_ttl       AgentRun core fields
  fields: state, plan_version, step_index,
          cost_usd, active_agent, started_at

run:{run_id}:plan:v{n}                      STRING  run_ttl       Plan JSON for version N
run:{run_id}:plan:current                   STRING  run_ttl       Pointer to current plan version

run:{run_id}:step:{step_id}:result          STRING  run_ttl       Step result JSON (append-only, never overwrite)
run:{run_id}:step:{step_id}:status          STRING  run_ttl       'PENDING' | 'RUNNING' | 'SUCCESS' | 'FAILED'

run:{run_id}:hypothesis:active              STRING  run_ttl       Current hypothesis JSON
  {hypothesis_text, confidence, supporting_evidence[], contradicting_evidence[]}

run:{run_id}:investigation:summary          STRING  run_ttl       Rolling investigation summary (<200 tokens)
run:{run_id}:scratchpad:{step_n}            STRING  run_ttl       Scratchpad JSON from step N
  {what_i_know, what_i_dont_know, hypotheses[], active_hypothesis,
   next_action_rationale, dead_ends[]}

run:{run_id}:context:summary               STRING  run_ttl       Compacted context window summary
run:{run_id}:agent:{agent_id}:state        HASH    run_ttl       Per-agent in-flight state

run:{run_id}:graph:service_context         STRING  10 min        Cached graph query for affected_service
run:{run_id}:graph:failure_patterns        STRING  10 min        Cached failure mode query
run:{run_id}:kb:results:{query_hash}       STRING  10 min        Cached KB search results

run:{run_id}:filesystem:index              STRING  run_ttl       Index of workspace files for this run
run:{run_id}:paused                        STRING  run_ttl       '1' if run is paused by user
run:{run_id}:stop_signal                   STRING  run_ttl       '1' if user requested stop

run:{run_id}:approval:{approval_id}:status STRING  approval_ttl  'PENDING' | 'APPROVED' | 'REJECTED'

# run_ttl = deadline_at + 24h
# approval_ttl = timeout_at + 1h
```

### Idempotency keys (tool call deduplication)

```
KEY                                         TYPE    TTL           DESCRIPTION
───────────────────────────────────────────────────────────────────────────────────────────

idem:{call_id}                              STRING  24h           Cached tool result for this call_id
  call_id = SHA256(run_id || step_index || tool_name || input_json)
```

### Circuit breaker state

```
KEY                                         TYPE    TTL           DESCRIPTION
───────────────────────────────────────────────────────────────────────────────────────────

cb:{tenant_id}:{tool_name}:state           STRING  none          'CLOSED' | 'OPEN' | 'HALF_OPEN'
cb:{tenant_id}:{tool_name}:failures        ZSET    60s (window)  Failure timestamps in sliding window
cb:{tenant_id}:{tool_name}:opened_at       STRING  none          When circuit was opened
```

### Rate limiting

```
KEY                                         TYPE    TTL           DESCRIPTION
───────────────────────────────────────────────────────────────────────────────────────────

rl:{tenant_id}:{tool_name}:{run_id}        INCR    60s           Call count this run for rate limiting
rl:model:{tenant_id}:{model}              INCR    60s           Model calls per minute
```

### SSE stream channels (per run)

```
KEY                                         TYPE    TTL           DESCRIPTION
───────────────────────────────────────────────────────────────────────────────────────────

stream:{run_id}:events                     STREAM  run_ttl + 1h  SSE event stream for UI
  fields per entry: event_type, agent_role, payload_json, ts
```

---

## 3. Neo4j — Knowledge Graph Schema

### Node labels and properties

```cypher
// ── SERVICE ──
(:Service {
  id:           STRING,   // uuid
  name:         STRING,   // "payment-service"
  tier:         STRING,   // "tier-1" | "tier-2" | "tier-3"
  sla_tier:     STRING,   // "gold" | "silver" | "bronze"
  repo_url:     STRING,
  tenant_id:    STRING,
  created_at:   DATETIME
})

// ── TEAM ──
(:Team {
  id:               STRING,
  name:             STRING,
  slack_channel:    STRING,
  oncall_schedule:  STRING,
  tenant_id:        STRING
})

// ── PERSON ──
(:Person {
  id:           STRING,
  name:         STRING,
  email:        STRING,
  role:         STRING,
  slack_id:     STRING,
  is_oncall:    BOOLEAN,
  tenant_id:    STRING
})

// ── FAILURE MODE ──
(:FailureMode {
  id:                   STRING,
  name:                 STRING,   // "connection_pool_exhaustion"
  description:          STRING,
  category:             STRING,   // "database" | "network" | "memory" | "config"
  avg_resolution_mins:  FLOAT,
  tenant_id:            STRING
})

// ── REMEDIATION ACTION ──
(:RemediationAction {
  id:                   STRING,
  name:                 STRING,   // "pgbouncer_reload"
  description:          STRING,
  command_id:           STRING,   // tool registry command_id
  risk_level:           STRING,
  success_rate:         FLOAT,    // rolling average
  avg_resolution_mins:  FLOAT,
  runbook_draft_id:     STRING,   // FK to kb_drafts
  tenant_id:            STRING
})

// ── INCIDENT ──
(:Incident {
  id:                   STRING,   // episode_id
  ticket_id:            STRING,
  run_id:               STRING,
  root_cause:           STRING,
  root_cause_category:  STRING,
  resolution_summary:   STRING,
  resolution_mins:      INT,
  outcome:              STRING,
  created_at:           DATETIME,
  tenant_id:            STRING
})

// ── KB DOCUMENT ──
(:KBDocument {
  id:               STRING,   // doc_id from Elasticsearch
  title:            STRING,
  source_type:      STRING,   // "runbook" | "sop" | "known_issue" | "faq"
  confidence:       FLOAT,
  last_validated:   DATE,
  tenant_id:        STRING
})

// ── ALERT GAP ──
(:AlertGap {
  id:             STRING,
  description:    STRING,
  service_name:   STRING,
  alert_type:     STRING,
  threshold:      STRING,
  detected_in:    STRING,   // run_id where gap was found
  status:         STRING,   // "OPEN" | "RESOLVED"
  tenant_id:      STRING
})
```

### Relationships

```cypher
// Ownership
(:Service)-[:OWNED_BY]->(:Team)
(:Team)-[:ON_CALL {since: DATETIME}]->(:Person)

// Service topology
(:Service)-[:DEPENDS_ON {
  criticality:      STRING,   // "hard" | "soft"
  dependency_type:  STRING    // "sync" | "async" | "data"
}]->(:Service)

// Failure patterns
(:Service)-[:HAS_KNOWN_FAILURE {
  count:              INT,
  last_seen:          DATETIME,
  avg_resolution_mins: FLOAT
}]->(:FailureMode)

// Incident relationships
(:Incident)-[:AFFECTED]->(:Service)
(:Incident)-[:CAUSED_BY]->(:FailureMode)
(:Incident)-[:RESOLVED_BY]->(:RemediationAction)

// Remediation to failure mode
(:RemediationAction)-[:FIXES {
  success_count: INT,
  total_attempts: INT
}]->(:FailureMode)

// Remediation to KB document
(:RemediationAction)-[:DOCUMENTED_IN]->(:KBDocument)

// Alert gaps
(:Service)-[:MISSING_ALERT]->(:AlertGap)
```

### Core queries used by agents

```cypher
// 1. Get full service context at run start
MATCH (s:Service {name: $service_name, tenant_id: $tenant_id})
OPTIONAL MATCH (s)-[:OWNED_BY]->(t:Team)
OPTIONAL MATCH (t)-[:ON_CALL]->(p:Person)
OPTIONAL MATCH (s)-[:DEPENDS_ON]->(dep:Service)
OPTIONAL MATCH (upstream:Service)-[:DEPENDS_ON]->(s)
OPTIONAL MATCH (s)-[rel:HAS_KNOWN_FAILURE]->(f:FailureMode)
OPTIONAL MATCH (f)<-[:FIXES]-(r:RemediationAction)
RETURN s, t, p,
       collect(DISTINCT dep) AS dependencies,
       collect(DISTINCT upstream) AS dependents,
       collect(DISTINCT {failure: f, frequency: rel.count, action: r}) AS known_failures
ORDER BY rel.count DESC

// 2. Seed hypotheses from failure patterns
MATCH (f:FailureMode)<-[rel:HAS_KNOWN_FAILURE]-(s:Service {name: $service_name, tenant_id: $tenant_id})
OPTIONAL MATCH (r:RemediationAction)-[:FIXES]->(f)
OPTIONAL MATCH (r)-[:DOCUMENTED_IN]->(doc:KBDocument)
RETURN f.name, f.description, rel.count, rel.avg_resolution_mins,
       collect(r.name) AS remediations,
       collect(doc.id) AS kb_doc_ids
ORDER BY rel.count DESC
LIMIT 5

// 3. Escalation routing
MATCH (s:Service {name: $service_name, tenant_id: $tenant_id})-[:OWNED_BY]->(t:Team)
MATCH (t)-[:ON_CALL]->(p:Person)
RETURN p.name, p.email, p.slack_id, t.slack_channel, t.oncall_schedule

// 4. Post-incident upsert (run by graph update job)
MERGE (i:Incident {id: $episode_id, tenant_id: $tenant_id})
SET i.ticket_id = $ticket_id,
    i.root_cause = $root_cause,
    i.root_cause_category = $root_cause_category,
    i.resolution_mins = $resolution_mins,
    i.created_at = datetime()
WITH i
MATCH (s:Service {name: $affected_service, tenant_id: $tenant_id})
MERGE (i)-[:AFFECTED]->(s)
WITH i, s
MERGE (f:FailureMode {name: $root_cause_category, tenant_id: $tenant_id})
MERGE (i)-[:CAUSED_BY]->(f)
MERGE (f)<-[rel:HAS_KNOWN_FAILURE]-(s)
SET rel.count = coalesce(rel.count, 0) + 1,
    rel.last_seen = datetime(),
    rel.avg_resolution_mins = (coalesce(rel.avg_resolution_mins, $resolution_mins) + $resolution_mins) / 2
WITH i, f
MERGE (r:RemediationAction {name: $resolution_action, tenant_id: $tenant_id})
MERGE (i)-[:RESOLVED_BY]->(r)
MERGE (r)-[fix:FIXES]->(f)
SET fix.success_count = coalesce(fix.success_count, 0) + 1,
    fix.total_attempts = coalesce(fix.total_attempts, 0) + 1
```

---

## 4. Elasticsearch — Knowledge Base Index

### Index: `kb_chunks_{tenant_id}`

```json
{
  "mappings": {
    "properties": {
      "chunk_id":       { "type": "keyword" },
      "doc_id":         { "type": "keyword" },
      "tenant_id":      { "type": "keyword" },

      "source_type":    { "type": "keyword" },
      "title":          { "type": "text", "analyzer": "standard" },

      "content":        {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },

      "content_vector": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine"
      },

      "metadata": {
        "properties": {
          "service":              { "type": "keyword" },
          "category":             { "type": "keyword" },
          "severity_applicable":  { "type": "keyword" },
          "last_validated":       { "type": "date" },
          "validated_by":         { "type": "keyword" },
          "confidence":           { "type": "float" },
          "prerequisites":        { "type": "text" },
          "tags":                 { "type": "keyword" }
        }
      },

      "chunk_position":   { "type": "integer" },
      "total_chunks":     { "type": "integer" },
      "prev_chunk_id":    { "type": "keyword" },
      "next_chunk_id":    { "type": "keyword" },

      "usage_stats": {
        "properties": {
          "times_retrieved":          { "type": "integer" },
          "times_used":               { "type": "integer" },
          "avg_helpfulness_score":    { "type": "float" },
          "last_retrieved":           { "type": "date" }
        }
      },

      "created_at":   { "type": "date" },
      "updated_at":   { "type": "date" }
    }
  },
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "standard": {
          "type": "standard",
          "stopwords": "_english_"
        }
      }
    }
  }
}
```

### Hybrid retrieval query (BM25 + dense vector + metadata filter)

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "{query_text}",
            "fields": ["content^2", "title^3"],
            "type": "best_fields"
          }
        }
      ],
      "filter": [
        { "term": { "tenant_id": "{tenant_id}" } },
        {
          "bool": {
            "should": [
              { "term": { "metadata.service": "{affected_service}" } },
              { "bool": { "must_not": { "exists": { "field": "metadata.service" } } } }
            ]
          }
        },
        { "range": { "metadata.confidence": { "gte": 0.6 } } }
      ]
    }
  },
  "knn": {
    "field": "content_vector",
    "query_vector": "{embedding_array}",
    "k": 20,
    "num_candidates": 100,
    "filter": [
      { "term": { "tenant_id": "{tenant_id}" } },
      { "range": { "metadata.confidence": { "gte": 0.6 } } }
    ]
  },
  "rank": {
    "rrf": { "window_size": 20 }
  },
  "_source": {
    "excludes": ["content_vector"]
  },
  "size": 10
}
```

---

## 5. Filesystem Workspace Structure

### Root: `s3://{bucket}/workspace/runs/{run_id}/`

```
workspace/runs/{run_id}/
│
├── reasoning/
│   ├── step_01.json          ← full scratchpad at step 1
│   ├── step_02.json
│   ├── ...
│   └── summary.md            ← rolling investigation summary (human-readable)
│
├── diagnostics/
│   ├── agent_a/              ← if parallel diagnosis agents
│   │   ├── service_a_check.json
│   │   └── pg_stat_activity.json
│   ├── agent_b/
│   │   └── service_b_metrics.json
│   └── combined.json         ← merged output from synthesis call
│
├── logs/
│   ├── pgbouncer.json        ← parsed + summarized log output
│   ├── user_api.json
│   └── {service}_{source}_raw.txt   ← full raw log (large, referenced by pointer)
│
├── tools/
│   ├── kb_search_01.json     ← raw KB search results before truncation
│   ├── past_incidents_01.json
│   └── graph_query_01.json
│
├── plan/
│   ├── v1.json               ← original plan
│   ├── v2.json               ← after first replan
│   └── v3.json               ← after second replan (rejection)
│
├── resolution/
│   ├── step_r1_result.json
│   ├── step_r2_result.json
│   └── credential_audit.json ← which credential was used, when, TTL (no credential values)
│
├── verification/
│   ├── step_r1_check.json
│   └── final_check.json
│
├── communication/
│   ├── user_update_01.md
│   ├── stakeholder_slack.json
│   └── escalation_summary.md
│
└── post_incident/
    ├── resolution_summary.md     ← narrative resolution for episodic memory
    ├── incident_record.json      ← structured, feeds knowledge graph update job
    ├── graph_updates.json        ← Cypher update operations for graph job
    ├── skill_feedback.json       ← skill gap records
    └── reasoning_quality.json    ← calibration report
```

### Scratchpad JSON schema (per step)

```json
{
  "step_index": 4,
  "agent_role": "DIAGNOSIS",
  "timestamp": "2024-01-15T23:14:02Z",
  "what_i_know": "string — established facts only",
  "what_i_dont_know": "string — gaps preventing certainty",
  "hypotheses": [
    {
      "id": "h1",
      "description": "string",
      "confidence": 0.72,
      "supporting_evidence": ["string"],
      "contradicting_evidence": ["string"],
      "status": "ACTIVE | DEAD | CONFIRMED"
    }
  ],
  "active_hypothesis": "h1",
  "dead_ends": ["string — approaches ruled out and why"],
  "next_action_rationale": "string — why this action reduces uncertainty most",
  "action": {
    "tool": "string",
    "command_id": "string",
    "input": {},
    "expected_outcome": "string",
    "confidence": 0.0,
    "fallback_if_wrong": "string"
  }
}
```

### Incident record JSON schema (feeds graph update job)

```json
{
  "run_id": "uuid",
  "episode_id": "uuid",
  "ticket_id": "string",
  "tenant_id": "uuid",
  "updates": [
    {
      "type": "UPSERT_INCIDENT",
      "data": {
        "incident_id": "uuid",
        "ticket_id": "string",
        "root_cause": "string",
        "root_cause_category": "string",
        "resolution_action": "string",
        "resolution_time_mins": 23,
        "affected_service": "string"
      }
    },
    {
      "type": "UPDATE_FAILURE_MODE_FREQUENCY",
      "data": {
        "failure_mode": "string",
        "service": "string",
        "increment_count": 1,
        "resolution_mins_sample": 23
      }
    },
    {
      "type": "UPSERT_REMEDIATION_ACTION",
      "data": {
        "action_name": "string",
        "runbook_draft_id": "uuid",
        "success": true,
        "resolution_time_mins": 23
      }
    },
    {
      "type": "FLAG_MISSING_ALERT",
      "data": {
        "service": "string",
        "alert_type": "string",
        "threshold": "string",
        "note": "string"
      }
    }
  ]
}
```

---

## 6. Quick Reference — All Table Names

```
PostgreSQL tables
─────────────────────────────────────────────────────────────────
tenants                   tenant configuration
teams                     service owning teams
persons                   team members, on-call tracking
agent_runs                one row per ticket run (orchestrator source of truth)
agent_steps               one row per executed step
tool_calls                idempotency + full tool call audit
approvals                 human approval requests and decisions
policy_audit              immutable policy decision log
episodes                  resolved incident memory (pgvector embeddings)
episode_steps             per-step detail for each episode
knowledge_gap_reports     KB and skill gaps found during runs
kb_drafts                 KB document drafts pending human review
skill_drafts              SKILL.md drafts pending human review
skills_usage              which skills were loaded and used per run
service_access_registry   per-service connection and auth config
harness_configs           versioned harness configuration snapshots
agent_instances           per-agent spawn records for a run
circuit_breakers          per-tool circuit breaker state

Redis key spaces
─────────────────────────────────────────────────────────────────
run:{run_id}:*            working memory for active run
idem:{call_id}            tool call idempotency cache
cb:{tenant}:{tool}:*      circuit breaker state
rl:{tenant}:{tool}:*      rate limit counters
stream:{run_id}:events    SSE event stream for UI

Neo4j node labels
─────────────────────────────────────────────────────────────────
Service                   microservices and APIs
Team                      owning teams
Person                    team members
FailureMode               known failure patterns
RemediationAction         known fixes
Incident                  resolved incidents (linked to episodes)
KBDocument                runbooks and SOPs in the KB
AlertGap                  missing monitoring coverage

Elasticsearch indexes
─────────────────────────────────────────────────────────────────
kb_chunks_{tenant_id}     knowledge base document chunks with embeddings

S3 paths
─────────────────────────────────────────────────────────────────
workspace/runs/{run_id}/  per-run workspace (reasoning, logs, plans, output)
```

---

## 7. Notes for Claude Code

Read these before writing any migration or schema code.

**Rule 1 — Only the orchestrator writes agent_runs.state.** Every other component reads it. No agent, gateway, or tool writes state directly. Enforce this with a PostgreSQL row-level security policy or a CHECK constraint on the update path.

**Rule 2 — agent_steps is append-only.** Steps are never updated after they are written. A retry creates a new step row with retry_count incremented. The original failed step stays as-is.

**Rule 3 — policy_audit is insert-only.** No UPDATE or DELETE on this table ever. Create a PostgreSQL role with INSERT-only permission and use it for the policy guard service.

**Rule 4 — Embeddings are computed outside PostgreSQL.** Call the embedding model, get the vector, then insert it. Never compute embeddings inside a PostgreSQL trigger or function.

**Rule 5 — Redis keys always include tenant_id prefix.** Use a key builder function. Never construct raw Redis keys as strings scattered through the codebase.

**Rule 6 — Filesystem paths are always constructed from run_id.** Never allow an agent to supply a file path directly. The harness builds the path from `workspace/runs/{run_id}/{subdirectory}`.

**Rule 7 — No credentials in any table.** service_access_registry stores references (ARN, vault path, SSM param name). Credentials live in the secret store only and are retrieved at runtime into memory-only handles.

**Rule 8 — Plans are versioned, never mutated.** When replanning, insert a new row in plan history and increment plan_version. Never UPDATE plan_json in place.

**Rule 9 — Graph updates run in a batch job, not inline.** The Knowledge Creator writes graph_updates.json to the filesystem. A separate nightly job reads these files and applies them to Neo4j. This prevents write conflicts from concurrent runs.

**Rule 10 — Episode embeddings use IVFFlat with lists=100.** Run ANALYZE on the table after bulk inserts. Rebuild the index monthly or when row count doubles.
