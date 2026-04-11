# Build Plan — Ticket Agent System

Build in this exact order. Complete each phase before starting the next.
Ask me to confirm before moving to the next phase.

---

## Phase 1 — Foundation (do this first)
Goal: orchestrator runs, state machine works, no LLM yet

1. Project scaffold: TypeScript, ESM, strict mode, path aliases
2. Database schema: run migrations for agent_runs, agent_steps, tool_calls, approvals
3. Redis client with key pattern helpers
4. Orchestrator state machine: IDLE → PLANNING → DISPATCHING → EXECUTING → REPLANNING → COMPLETING → DONE → FAILED
5. AgentRun record: create, read, update state (orchestrator only)
6. Basic API: POST /runs (create run), GET /runs/:id (read run)

Deliverable: I can POST a ticket and watch it move through states

---

## Phase 2 — Model Plane
Goal: LLM generates a structured plan

1. Anthropic SDK wrapper with retry, cost tracking, structured output
2. Model router: picks model based on ticket priority
3. Plan generator: takes ticket + context → returns JSON plan
4. Plan schema validation: reject malformed output, retry with correction prompt
5. Replanner: takes failed plan + failure reason → returns revised plan
6. Store plans in agent_runs.plan_json, versioned

Deliverable: POST /runs produces a real plan from Claude

---

## Phase 3 — Policy Guard
Goal: nothing executes without passing policy

1. Policy guard service: evaluates tool call requests
2. Authority matrix: which agent role can call which tool
3. Confidence gate: per tool risk level and threshold
4. Budget gate: check cost_so_far vs max_cost_usd
5. Circuit breaker: per tool, sliding window failure rate
6. Input sanitizer: schema validation + forbidden patterns
7. Scope check: tenant isolation
8. All gates return {allowed, reason, required_approvals}

Deliverable: every tool call request goes through policy, nothing bypasses it

---

## Phase 4 — Tool Plane / MCP Gateway
Goal: tools can execute safely

1. Tool registry: manifest.json with all tool definitions
2. MCP gateway: receives tool call, calls policy guard, executes, returns normalized result
3. Context resolver: (service, environment) → execution context with credential handle
4. Server access registry: database table with connection methods per service
5. Execution runner: SSH_BASTION, SSM, K8S_EXEC, API handlers
6. Command sanitizer: template-based commands, no raw bash from LLM
7. Output sanitizer: strip credentials, internal IPs before returning
8. Tool call log: write every call to tool_calls table

Implement these tools first:
- get_ticket_details
- search_knowledge_base  
- search_past_incidents
- run_command (with command_id, no raw bash)
- get_server_logs
- update_ticket
- notify_user (stub)
- request_human_approval (creates approval record, suspends run)
- escalate (stub)

Deliverable: agent can SSH to a server and read logs safely

---

## Phase 5 — Memory Layer
Goal: agents can read and write all 5 memory layers

1. Working memory: Redis key helpers, append-only step results
2. Episodic memory: PostgreSQL episodes table, pgvector similarity search
3. Semantic KB: Elasticsearch index, hybrid BM25 + dense vector search
4. Knowledge graph: Neo4j schema, basic Cypher queries
5. Filesystem workspace: S3 or local for runs, structured directory
6. Context assembler: harness function that builds context window from all layers

Deliverable: new run gets top-5 similar past incidents injected into plan prompt

---

## Phase 5b — Graph Ontology (add after basic graph works)

1. ServiceType nodes — type hierarchy for services (PostgreSQL IS-A RelationalDatabase IS-A DataStore)
2. FailureModeType nodes — failure mode hierarchy
3. IS_A relationships between type nodes
4. HAS_TYPE relationships from Service/FailureMode nodes to their type
5. Five inference rule queries (blast radius, co-failure, remediation chain,
   deployment correlation, type generalisation)
   — wrap each in a named function in src/graph/inference.ts
   — orchestrator calls the relevant ones at run start and during replanning
6. Synonym groups table and seed data
7. Normalise function — called before every KB search and graph query
8. Synonym management API (GET/POST /v1/admin/synonyms)
9. Seed the type hierarchy from seeds/service-types.yaml and seeds/failure-mode-types.yaml

---

## Phase 6 — Agent Roles
Goal: 6 distinct agent roles run in the correct sequence

1. Triage Agent: reads ticket, produces structured triage output
2. Diagnosis Agent: runs diagnostic loop, writes scratchpad, updates hypothesis
3. Resolution Agent: executes plan steps (all through policy guard)
4. Verification Agent: confirms each step worked
5. Communication Agent: drafts and sends stakeholder updates
6. Post-Incident Agent: writes to episodic memory and knowledge graph

Agent runner: spawns correct agent per orchestrator state
Parallel execution: diagnosis agents can run in parallel on multiple services
Handoff: agents never call each other — always through orchestrator

Deliverable: full run from ticket to resolution with all 6 agents

---

## Phase 7 — Skills Layer
Goal: skills load contextually, skillless runs work safely

1. Skills registry: skills/registry.json maps ticket types to skill files
2. Skill loader: reads SKILL.md files, injects into system prompt
3. Progressive disclosure: only load skills for current state
4. Fallback: general-investigation skill for unknown categories
5. Skillless run mode: all writes require human approval, broad observation mode
6. Skill usage tracking: log which skills were loaded vs actually used

Create these skills first:
- general-investigation (fallback)
- db-connection-management
- api-latency-diagnosis
- escalation

Deliverable: known ticket type loads skill, unknown ticket runs in safe fallback mode

---

## Phase 8 — Reasoning Loop (skillless)
Goal: agent can reason from first principles when no skill exists

1. Scratchpad schema: what_i_know, what_i_dont_know, hypotheses[], active_hypothesis, dead_ends
2. Exploratory reasoning call: produces scratchpad before any tool call
3. Re-reasoning triggers: contradiction signal, confidence plateau, verification failure
4. Investigation summary: rolling 200-token summary updated after each step
5. Three-level memory: last 2 scratchpads in context, summary in system prompt, full history on filesystem
6. Exit conditions: confidence threshold, escalation exit, budget exit, contradiction loop exit

Deliverable: agent with no skill can investigate an unknown ticket and either solve it or escalate with a full investigation summary

---

## Phase 9 — Knowledge Creator
Goal: every resolved ticket makes the system smarter

1. Episode enrichment: hypothesis trace, diagnostic usefulness, confidence history
2. KB gap detector: identifies missing or stale documents
3. KB draft generator: creates SKILL.md or runbook draft from resolution artifacts
4. Graph updater: Cypher update jobs from incident_record.json
5. Reasoning quality report: calibration scoring, optimal diagnostic sequence
6. Human review queue: drafts go to pending_review, never auto-publish
7. Skill feedback loop: gap records feed skill creation pipeline

Deliverable: after 3 test runs, system has generated at least 1 KB draft and 1 skill gap record

---

## Phase 10 — Observability
Goal: every event is traced and auditable

1. Distributed tracing: OpenTelemetry spans per agent, per tool call, per model call
2. Span attributes: run_id, step_index, agent_role, cost_usd, duration_ms
3. Audit log: immutable append-only table for all policy decisions and approvals
4. Cost tracking: per model call, per tool call, per run
5. RAG quality: log query, top-k results, whether used
6. Offline eval harness: test suite of historical tickets with known resolutions

Deliverable: I can see full trace for any run in console and audit log

---

## Phase 11 — UI
Goal: end user can watch agent work and approve actions

Build the UI from the spec in docs/12-ui-spec.md.
The UI design is already defined — implement it, do not redesign it.

1. SSE stream: server sends events for every agent action
2. Live terminal panel: streams sanitized command output
3. Hypothesis strip: updates in real time
4. Audit trace panel: right side, expandable rows
5. Timeline: bottom, event dots
6. Approval modal: fires when agent hits requires_human_approval
7. Replan card: fires when replan triggered, shows old vs new plan
8. Controls: Stop, Pause, Approve, View Full Log

Deliverable: UI matches the design in docs/12-ui-spec.md exactly

---

## Phase 12 — Integration & Testing
1. End-to-end test: real ticket → triage → diagnosis → resolution → verification → close
2. Failure tests: SSH timeout → replan, cost ceiling → pause, contradiction loop → escalate
3. Shadow mode: run new version in parallel, compare plans
4. Load test: 10 concurrent P1 tickets

---

## What to ask me before each phase
Before starting a phase:
1. Confirm the deliverable matches what I described
2. Tell me which files you will create or modify
3. Ask if there are any constraints I have not mentioned