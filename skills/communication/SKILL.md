---
name: stakeholder-communication
description:
  Drafts clear, appropriately scoped messages for users, teams, and
  stakeholders. Covers update messages, approval requests, credential
  requests, escalation alerts, and resolution notifications.
license: MIT
metadata:
  author: ops-agent
  version: '1.0.0'
---

## Communication principles

- One message, one ask — never combine status update with credential request
- Match urgency to tone — P1 Slack = urgent, P4 email = calm
- Never include technical detail the recipient does not need
- Always include: what action is needed and by when

---

## Message templates

### User/reporter update (mid-incident)

```
Subject: [INC{number}] Update — {symptom} — investigation in progress

Hi {reporter_name},

We have received your report and are actively investigating
{symptom} affecting {service_name}.

Current status: {what we know — one sentence}
Next update when: {specific condition}

Reference: {ticket_id} | Live status: {run_url}
```

---

### Credential request to target system team

```
To: {team_slack_channel}
@{oncall_person} — Action needed: credential update for {app_name}

Our integration {mule_app_name} is receiving HTTP 401 errors
from {endpoint_url} since {first_error_time}.

Impact: {impact — one sentence}

We need:
1. Confirmation whether credentials were recently rotated
2. If rotated: new credentials via secure handoff → {handoff_url}

Response needed within {30 min P1 | 2 hrs P2}.
Incident: {ticket_id}
```

**Never:** send credentials via Slack/email — always secure handoff URL only.

---

### P1 escalation alert

```
🚨 P1 — {service_name} | {symptom}

Started: {time} ({elapsed} minutes ago)
Impact: {user_impact}
Ticket: {ticket_id} | Trace: {run_url}

Finding: {hypothesis — one sentence} (confidence: {score})

Needed from {team}: {specific ask}
Response by: {time}

Agent run is paused and waiting.
```

---

### Resolution notification

```
Subject: [INC{number}] RESOLVED — {symptom}

{service_name} has been restored.

Root cause: {plain English, one sentence}
Fix: {what was done}
Restored at: {time} | Duration: {total}

No action needed from you.
New issues → open a new incident ticket.
```

---

### Approval request

```
To: {approver_slack}

⚠ Approval needed — {ticket_id}

The agent wants to: {plain English action}
Why: {one sentence reason}
Risk: {MEDIUM/HIGH} | Downtime: {if any} | Rollback: {Yes/No}

Approve or reject: {approval_url}
Timeout: {time} — auto-escalates if no response
```

---

## Channel selection

| Situation | Channel |
|---|---|
| P1 war room | #incidents-p1 + PagerDuty |
| Team escalation | {team_slack_channel} |
| Reporter update | ServiceNow comment (customer-visible) |
| Internal note | ServiceNow work note |
| Credential request | Slack DM to on-call + team channel |
| Resolution | ServiceNow comment + reporter email |
| Approval request | Slack DM to approver |

---

## Tone calibration

| Situation | Tone |
|---|---|
| P1 production revenue | Urgent, direct, no pleasantries |
| P2 production degraded | Professional, clear, concise |
| P3/P4 non-production | Friendly, not alarming |
| Credential request | Professional, deadline-specific |
| Resolution | Positive, brief, reassuring |

---

## Do not

- Include stack traces or raw log output in user messages
- Use internal system names (Redis, pgbouncer) in reporter updates — say "database layer"
- Send more than one update per 15 minutes unless priority escalates
- CC people who are not impacted or needed to act
- Ask for credentials via email or Slack — always secure handoff process only