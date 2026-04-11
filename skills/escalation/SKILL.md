---
name: escalation-decision-routing
description:
  Governs when to escalate, who to escalate to, and what information to
  include. Covers P1 war-room triggers, team routing, approval escalation,
  and graceful human handoff.
license: MIT
metadata:
  author: ops-agent
  version: '1.0.0'
---

## When to escalate

**Immediately — do not attempt diagnosis first:**
- Ticket mentions data loss, data corruption, or data exfiltration
- Ticket mentions security breach or unauthorized access
- Three or more P1 services simultaneously affected
- Regulated scope (financial, PII) with uncertain data integrity

**After diagnosis fails:**
- 8 diagnostic steps completed, confidence still below 0.50
- Contradiction loop: two diagnostics give conflicting results
- No tool access to the affected system
- Root cause found but fix requires access not available to agent

**Mid-resolution:**
- A remediation step made things worse (degradation detected)
- Human rejected fix and no safe alternative exists
- Cost budget exhausted before resolution

**On timeout:**
- Approval not received within window
- Credential update: P1 no response in 30 min, P2 no response in 2 hrs

---

## Who to escalate to

```
Tool: query_knowledge_graph
Query: escalation_routing
Input: { service: "{affected_service}", priority: "{priority}" }
```

Returns: team name, Slack channel, on-call person, PagerDuty schedule.

By category:
- integration → Integration platform / Mule team
- database → Database platform team
- network → Network / infrastructure team
- security → Security team + CISO (P1)
- application → Service owning team
- infrastructure → Cloud / DevOps team
- data corruption → Service team + Data governance + Legal (if PII)

By urgency:
- P1: Page on-call via PagerDuty + post in #incidents-p1
- P2: Slack to team channel + follow-up task
- P3/P4: Slack message only, no paging

---

## Escalation package

1. WHAT: One sentence symptom summary
2. WHEN: Exact start time from logs
3. IMPACT: Services affected, user impact estimate
4. TRIED: Steps taken, ruled out, confirmed
5. STATE: Agent hypothesis and confidence score
6. ASK: Specific ask (credentials / access / decision / approval)
7. LINK: Full agent trace URL
8. TICKET: ServiceNow ticket number

---

## Credential escalation

When 401/403 confirmed and new credentials needed:
- Escalate to target system team via Slack
- Include: app name, endpoint URL, first failure time, secure handoff link
- SLA: P1 → 30 min, P2 → 2 hrs
- After credentials provided: human stores in vault, approves update
- Agent executes update with vault reference only — never raw value

---

## Graceful handoff

1. Write investigation summary to filesystem
2. List every hypothesis tried including dead ends
3. List what access or information is needed to proceed
4. Set ticket state: awaiting human input
5. Keep run open and resumable

---

## Do not

- Page without a clear specific ask
- Escalate without the investigation summary
- Escalate to multiple teams without a clear primary owner
- Close the run on escalation
- Include credential values in any escalation message