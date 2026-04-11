---
name: ticket-triage-classification
description:
  Classifies incoming tickets by priority, category, and affected system.
  Extracts structured signal from unstructured ticket text. Selects the
  right skill set for downstream agents. Always the first skill loaded.
license: MIT
metadata:
  author: ops-agent
  version: '1.0.0'
---

## Purpose

Your only job in this skill is to produce a structured triage output.
Do not diagnose. Do not suggest fixes. Do not call write tools.
Read the ticket. Classify it. Output structured data. Done.

---

## Input you will receive

- Raw ticket title and description
- Source system metadata (ServiceNow fields, priority, category)
- Top 3 similar past incidents from episodic memory (if found)
- Affected service names (if extractable)

---

## What to produce

Output this exact JSON schema. Every field is required.

```json
{
  "priority":              "P1|P2|P3|P4",
  "priority_confidence":   0.0,
  "category":              "string",
  "category_confidence":   0.0,
  "affected_services":     ["string"],
  "affected_environment":  "production|staging|development|unknown",
  "symptom_type":          "string",
  "skill_recommendation":  "string",
  "skill_confidence":      0.0,
  "is_known_issue":        true|false,
  "known_issue_ref":       "string|null",
  "requires_immediate_escalation": true|false,
  "escalation_reason":     "string|null",
  "information_gaps":      ["string"],
  "routing_decision":      "DIAGNOSE|ESCALATE|KNOWN_FIX|MORE_INFO_NEEDED"
}
```

---

## Priority classification

**P1 — Critical** (classify as P1 if ANY of these are true):
- Production system completely down — zero functionality
- Revenue-impacting: payments failing, orders not processing, billing broken
- Data loss occurring or at risk
- Security breach or active attack
- > 50% of users affected
- SLA breach imminent (< 30 minutes)
- Multiple services simultaneously down (cascade failure)

**P2 — High** (classify as P2 if ANY of these are true):
- Production system degraded but partially functional
- Significant user impact but workaround exists
- Single critical service down, others functional
- Performance severely degraded (> 5x normal response time)
- Integration broken affecting downstream but not causing immediate revenue loss
- Error rate > 5% but < 50%

**P3 — Moderate**:
- Non-critical feature broken
- Small subset of users affected
- Degraded performance but within acceptable range
- Issue exists but auto-recovery is likely

**P4 — Low**:
- Cosmetic or minor issues
- No user impact
- Can wait for next business day

---

## Category classification

Use EXACTLY one of these categories. Do not invent new ones.

```
integration        — API-to-API, middleware (Mule, MQ), data pipeline
database           — Connection issues, slow queries, replication
network            — Connectivity, DNS, firewall, TLS/SSL
application        — App crash, memory, config error, bad deployment
security           — Auth failure, certificate, access control
infrastructure     — Server, container, Kubernetes, cloud resource
performance        — Latency, throughput, CPU/memory pressure
data               — Bad data, transform failure, validation
monitoring         — Alert misconfiguration, false positive
general            — Cannot classify with confidence > 0.6
```

---

## Skill recommendation

Map ticket category + affected system to the right skill.
Output the skill file path as `skill_recommendation`.

```
Keyword match → skill_recommendation
─────────────────────────────────────────────────────────
mule/anypoint/cloudhub/integration API    → mule-api-connectivity
database/postgres/pgbouncer/connection    → db-connection-management
latency/timeout/response time/503/504     → api-latency-diagnosis
auth/401/403/credential/certificate       → mule-api-connectivity (if Mule)
                                            or auth-failure-diagnosis (if not)
SAP/ABAP/IDoc/BAPI/RFC                    → sap-integration
unknown/cannot classify                   → general-investigation (fallback)
```

If `skill_confidence` < 0.60: set `routing_decision = MORE_INFO_NEEDED`
and list what information is missing in `information_gaps`.

---

## Known issue detection

Before generating a diagnosis plan, check if this is already a documented known issue.

```
Tool: search_knowledge_base
Query: "{symptom_type} {affected_service} known issue"
Filter: source_type = "known_issue"
```

If a known issue matches with confidence > 0.75:
- Set `is_known_issue = true`
- Set `known_issue_ref` to the document title
- Set `routing_decision = KNOWN_FIX`
- The orchestrator will skip full diagnosis and go directly to the fix

---

## Immediate escalation triggers

Set `requires_immediate_escalation = true` AND `routing_decision = ESCALATE` if:

- Ticket explicitly says "data loss" or "data corruption" — never attempt automated fix
- Ticket mentions "security breach", "unauthorized access", "data exfiltration"
- Ticket affects more than 3 services simultaneously
- Ticket says the issue is already known and a human is working it
- The system has no read access to affected services (cannot diagnose blind)

---

## Information gaps

List anything missing that would change the classification or routing:

Examples:
- "Affected service name not specified — cannot route to correct skill"
- "Environment not stated — cannot confirm if this is production"
- "Error message not included — cannot narrow category"
- "Ticket says 'API broken' but no API name given"

If `information_gaps` is non-empty, spawn Communication Agent to request
the missing information from the ticket reporter before diagnosis begins.

---

## Do not

- Diagnose or run any tool calls beyond search_knowledge_base
- Call any write tools
- Modify the ticket in the source system
- Make assumptions about environment (default to 'unknown' if not stated)
- Set priority higher than P2 without explicit evidence of P1 criteria