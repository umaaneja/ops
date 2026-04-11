---
name: knowledge-base-search
description:
  How to query the knowledge base effectively. Covers query construction,
  result interpretation, when to search vs. episodic memory, and how to
  know when a result is actually useful.
license: MIT
metadata:
  author: ops-agent
  version: '1.0.0'
---

## When to search the KB

Search KB for:
- Step-by-step remediation procedures
- How a service or system works
- Known issue documentation
- SOPs for specific incident types
- Architecture context for unfamiliar services

Do NOT search KB for:
- Current system state → use monitoring tools
- Whether a past incident happened → use episodic memory
- Who owns a service → use knowledge graph
- Real-time metrics → use monitoring API

---

## Query construction

**Bad:** "database issue" — too vague, returns everything
**Good:** "pgbouncer connection pool exhaustion payment-service remediation"

**Formula:**
```
{failure_mode_canonical} + {service_name} + {what_you_want}
```

**Examples:**
- "mule api 401 credential rotation procedure"
- "connection pool exhaustion postgres restart steps"
- "timeout configuration mulesoft anypoint increase"
- "OOM out of memory mule worker heap increase"

**Use canonical terms, not ticket language:**
- "the thing keeps dying" → "application crash OOM memory"
- "auth is broken" → "authentication failure 401 unauthorized credential"
- "it's slow" → "high latency timeout response time degraded performance"

---

## Evaluating a result

**Useful if:**
- Service matches or is same type as affected service
- Failure mode matches active hypothesis
- Confidence ≥ 0.70
- Last validated within 90 days

**Not useful if:**
- Different service with no architectural overlap
- Confidence < 0.60 — treat as unreliable, do not cite
- Last validated > 180 days — flag as stale, verify before using
- Describes symptom but not cause

---

## Search sequence

```
1. Search: exact failure mode + service name
   → Confidence ≥ 0.75? Use it, stop searching.

2. Search: failure mode + service type (generalise)
   → "pgbouncer pool" → "connection pool exhaustion database"

3. Search: symptom + resolution keyword
   → "401 unauthorized fix integration API"

4. Search: known issues for this service
   → "{service_name} known issue recurring"

5. After 3 searches with no useful result:
   → Flag KB gap in scratchpad
   → Continue with first-principles reasoning
   → Post-incident: Knowledge Creator generates draft runbook
```

---

## Do not

- Use KB results with confidence < 0.60 as evidence
- Run more than 5 KB searches per diagnostic run
- Repeat the same search with minor wording changes
- Trust KB results that contradict strong live diagnostic evidence