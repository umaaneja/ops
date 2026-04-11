# Claude Code Session Guide

## 1. How to Start Each Claude Code Session

### Session 1 — Phase 1

Open your terminal in the project folder and start Claude Code:

```bash
claude
```

Then type:

```
Read CLAUDE.md and BUILDPLAN.md first.
Then read all files in the docs/ folder in order from 00 to 13.
Tell me what you understood about the system before writing any code.
Then start Phase 1 from BUILDPLAN.md.
```

Wait for Claude Code to confirm its understanding before it writes code. If it misses something, correct it before it starts.

---

### Session 2 and Beyond

At the start of every new Claude Code session, type this:

```
Read CLAUDE.md and BUILDPLAN.md.
We completed Phase [N].
Now start Phase [N+1].
Before writing code, tell me your plan for this phase.
```

Always make it confirm its plan before writing. This prevents it from going in the wrong direction.

---

## 2. The Best Prompt Patterns for Claude Code

### Pattern 1 — Before Any New Component

```
Before writing this component, answer these questions:
1. Which layer of the architecture does this belong to?
2. What does it receive as input and what does it return?
3. What does it never do (constraints)?
4. Which other components does it interact with?
Then write the code.
```

---

### Pattern 2 — When It Gets Something Wrong

```
This is wrong because [reason].
The spec in docs/[file].md says [exact quote].
Rewrite this to match the spec.
Do not change anything else.
```

---

### Pattern 3 — Policy Guard

```
Write the policy guard.
It must be pure deterministic code — no LLM involved.
Every tool call request must pass through it.
It evaluates these gates in order: [list from docs/10-policy-guard.md].
It returns {allowed: bool, reason: string, required_approvals: string[]}.
Nothing in the system can bypass it.
Write tests first, then the implementation.
```

---

### Pattern 4 — Orchestrator

```
Write the orchestrator state machine.
It is the ONLY component that writes to agent_runs.state.
No agent, no tool, no gateway writes state directly.
The valid transitions are: IDLE → PLANNING → DISPATCHING → EXECUTING → REPLANNING → COMPLETING → DONE and any state → FAILED.
Invalid transitions must throw and be logged.
```

---

### Pattern 5 — When a Phase Is Complete

```
Phase [N] is done.
Before we move to Phase [N+1]:
1. Run all tests and show me results
2. Show me the file tree of what was created
3. List any decisions you made that were not in the spec
4. List any things in the spec you did not implement yet and why
```

---

## 3. Files to Create Right Now

### Step 1 — Create docs/ Files

Copy the relevant sections into the following structure:

```
Our conversation section               → docs/ file
─────────────────────────────────────────────────────
Core Mental Model + harness spec      → 00-overview.md + 02-harness.md
Control plane spec                   → 01-control-plane.md
Multi-agent design                  → 03-agents.md
Model plane spec                    → 04-model-plane.md
Tool plane spec                     → 05-tool-plane.md
Memory management spec              → 06-memory.md
Server access spec                  → 07-server-access.md
Reasoning loop spec                 → 08-reasoning.md
Knowledge creator spec              → 09-knowledge-creator.md
Policy guard spec                   → 10-policy-guard.md
Observability spec                  → 11-observability.md
Terminal UI + audit trace spec      → 12-ui-spec.md
Data models (all schemas)           → 13-data-models.md
```

---

### Step 2 — Run Claude Code

```bash
cd ticket-agent
claude
```

Paste this as your first message:

```
I have set up the docs/ folder with the full architecture spec.
Read CLAUDE.md first, then read every file in docs/ in order.
After reading, summarize back to me in 10 bullet points what you understood.
Do not write any code yet.
```

Once it summarizes correctly, say:

```
Good. Now start Phase 1 from BUILDPLAN.md.
Show me your plan before writing anything.
```

---

## 4. Hard Rules (Critical)

Paste this at the start of Phase 3 and Phase 6:

```
Hard rules — never break these:

1. The LLM never calls a tool. It produces a tool call request (JSON).
   The orchestrator reads it. The policy guard evaluates it.
   The MCP gateway executes it. These are three separate code paths.

2. Never use eval(), exec(), or shell string interpolation with LLM output.
   All commands are template-based with parameters filled from service_metadata.

3. Credentials never appear in logs, working memory, or any response to the LLM.
   They are opaque handles stored only in execution context, in memory, during the run.

4. Plans are versioned. agent_runs.plan_json is never mutated.
   Replanning creates plan_v2, plan_v3. Old versions stay for audit.

5. Working memory in Redis is append-only within a run.
   Step results are never overwritten.

6. The knowledge base is never agent-writable.
   Only episodic memory and the knowledge graph accept agent writes, at run end only.
```

---

## 5. Notes

* Always validate Claude's understanding before code generation
* Keep strict separation of concerns across architecture layers
* Enforce deterministic execution paths for safety-critical components
* Maintain full auditability of plans and state transitions
