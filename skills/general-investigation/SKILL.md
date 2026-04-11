---
name: general-investigation
description:
  Fallback skill for unknown ticket types with no matching runbook. Covers
  read-only diagnostics, hypothesis formation across broad system checks,
  and escalation when confidence does not rise after six diagnostic steps.
license: MIT
metadata:
  author: ops-agent
  version: '1.0.0'
---

## When to load this skill
Load when: no specific skill matched this ticket type.
This is the fallback skill for unknown situations.

## Behaviour in skillless mode
- You are in unknown territory. No runbook exists for this ticket type.
- Default to read-only operations until hypothesis confidence exceeds 0.75.
- All write operations require human approval regardless of confidence score.
- Document every observation to the filesystem before forming any hypothesis.
- Prefer broad system-state checks over targeted queries.
- Escalate rather than guess if confidence does not rise after 6 diagnostic steps.

## Diagnostic sequence
1. Identify the affected service and its dependencies from the knowledge graph
2. Check recent error rates and latency metrics (monitoring API)
3. Check recent deployments and config changes in the last 2 hours
4. Read application logs for the affected service (last 30 minutes)
5. Check if other services are also degraded (blast radius check)
6. Form hypothesis based on observations — do not form before step 4

## Do not
- Execute any remediation before hypothesis confidence is above 0.75
- Call write tools without human approval
- Repeat the same diagnostic more than twice
- Assume the first hypothesis is correct without confirming evidence