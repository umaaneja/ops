---
name: post-incident-learning
description:
  Writes the resolution summary, updates episodic memory, queues knowledge
  graph updates, and identifies what the system should learn from this
  incident. Runs after every resolved or escalated run.
license: MIT
metadata:
  author: ops-agent
  version: '1.0.0'
---

## What to produce (four artifacts)

---

### 1. Resolution summary (human-readable)

Path: `workspace/runs/{run_id}/post_incident/resolution_summary.md`
Also written to ServiceNow as a work note.

```markdown
# Incident Resolution Summary

**Ticket:** {ticket_id}  
**Service:** {affected_service}  
**Duration:** {start} → {end} ({total_minutes} min)

## Root Cause
{One paragraph — plain English — what caused it and why}

## Timeline
{start_time} — Issue began  
{detection_time} — Alert / ticket created  
{diagnosis_time} — Root cause confirmed ({confidence}%)  
{resolution_time} — Fix applied  
{verification_time} — Service restored  

## What Was Done
1. {step — plain English}
2. {step}

## What Was Ruled Out
- {hypothesis} — ruled out because {reason}

## Recurrence Prevention
{What should be done to prevent this — monitoring gaps, config changes}

## Agent Run Cost
${cost} | {minutes} minutes to resolve
```

---

### 2. Incident record (structured JSON → knowledge graph)

Path: `workspace/runs/{run_id}/post_incident/incident_record.json`

```json
{
  "ticket_id": "string",
  "affected_service": "string",
  "root_cause": "string",
  "root_cause_category": "string (canonical)",
  "resolution_action": "string (command_id)",
  "resolution_time_mins": 0,
  "outcome": "RESOLVED|ESCALATED|FAILED",
  "confidence_at_resolution": 0.0,
  "hypotheses_tried": [
    { "hypothesis": "string", "was_correct": true, "dead_end": false }
  ],
  "tools_that_helped": ["string"],
  "tools_that_did_not_help": ["string"],
  "kb_chunks_used": ["chunk_id"],
  "was_skillless": false
}
```

---

### 3. Skill feedback (gaps → human review queue)

Path: `workspace/runs/{run_id}/post_incident/skill_feedback.json`

```json
[
  {
    "gap_type": "MISSING_SKILL|MISSING_DOCUMENT|STALE_DOCUMENT",
    "description": "what was missing and why it mattered",
    "affected_service": "string",
    "what_agent_figured_out_manually": ["string"],
    "estimated_steps_saved_if_existed": 0,
    "priority": "HIGH|MEDIUM|LOW"
  }
]
```

Common gaps to look for:
- Steps agent had to reason from scratch (no KB match)
- Error messages unfamiliar to the skill
- Escalation routing that needed correction

---

### 4. Graph updates

Path: `workspace/runs/{run_id}/post_incident/graph_updates.json`

Always include:
```json
[
  { "type": "UPSERT_INCIDENT", "data": { ... } },
  { "type": "UPDATE_FAILURE_MODE_FREQUENCY", "data": { ... } },
  { "type": "LINK_ACTION_TO_FAILURE_MODE", "data": { ... } }
]
```

If monitoring gap detected (issue detectable earlier with better alert):
```json
{ "type": "FLAG_MISSING_ALERT", "data": {
    "service": "string",
    "alert_type": "string",
    "threshold": "string"
}}
```

---

## What makes a good summary

**Good root cause:**
"The Mule CloudHub application ran out of Java heap memory due to a memory leak
introduced in deployment v2.4.1 at 06:02. Heap grew steadily from 6:02 until
all workers were killed at 6:14."

**Bad root cause:**
"Application crashed due to OOM."

**Good recurrence prevention:**
"Add heap usage alert at 75% for sap-order-integration-prod. Investigate the
memory leak in v2.4.1 — suspect batch size increase without corresponding pool
size increase."

**Bad recurrence prevention:**
"Restart the app if it crashes again."

---

## Do not

- Leave hypotheses_tried empty — even if the first was correct
- Skip skill_feedback — this is how the system gets smarter
- Use invented data in graph updates — confirmed facts only
- Close the ServiceNow ticket without human review for P1/P2