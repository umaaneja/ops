# Skill: Mule API Connectivity
# File: skills/mule-api-connectivity/SKILL.md
# Version: 1.0.0
# Author: Platform Integration Team
#
# NAME: Mule API Connectivity Diagnostic
# DESCRIPTION: Diagnoses MuleSoft Anypoint Platform API failures including
#   credential errors, connection timeouts, app crashes, flow failures,
#   and connector misconfigurations. Covers CloudHub and on-premise runtimes.
#   Does NOT cover SAP-specific layers — use skill: sap-integration for those.

---

## When to load this skill

Load when ticket contains ANY of:

**Keywords:** mule · mulesoft · anypoint · cloudhub · mule flow · mule app ·
mule connector · mule runtime · dataweave · mule worker · mule API

**Symptoms:** API not responding · integration broken · data not flowing ·
connector error · flow failing · mule stopped · mule crashed · 401 from mule ·
403 from mule · connection refused on port 8081 · timeout calling integration API

**Do NOT load this skill for:**
- Pure SAP issues with no Mule involvement → use sap-integration
- Pure REST API issues with no Mule involvement → use api-latency-diagnosis
- Database issues → use db-connection-management

---

## Architecture this skill covers

```
[Client / Source System]
        │  HTTP / HTTPS / SFTP / JMS
        ▼
[Anypoint Platform]
  ┌──────────────────────────────────────────────────┐
  │  Mule Application (CloudHub or On-Premise)       │
  │  ┌──────────┐  ┌────────────┐  ┌─────────────┐  │
  │  │ Listener │→ │ DataWeave  │→ │  Connector  │  │
  │  │ HTTP/    │  │ Transform  │  │  HTTP/DB/   │  │
  │  │ SFTP/JMS │  │            │  │  SAP/SFTP   │  │
  │  └──────────┘  └────────────┘  └─────────────┘  │
  └──────────────────────────────────────────────────┘
        │  Outbound call (authenticated)
        ▼
[Target System]  e.g. ERP, CRM, DB, third-party API
```

**Failure points this skill diagnoses:**
- A: Mule app not running (crashed, OOM, bad deploy)
- B: Wrong credentials for target system (401/403 from target)
- C: Connection timeout to target (network, firewall, slow target)
- D: DataWeave transform error (bad input data)
- E: Connector misconfiguration (wrong URL, wrong port)
- F: Anypoint platform issue (CloudHub infrastructure)

---

## Diagnostic sequence

Always work top to bottom. Confirm each layer before going deeper.

---

### Step 1 — Is the Mule app running?

```
Tool: query_monitoring_api
Source: anypoint_cloudhub
Action: get_application_status
Input: { app_name: "{mule_app_name}", environment: "{environment}" }
```

**Interpret:**

| Status | Meaning | Next step |
|---|---|---|
| STARTED + workers > 0 | App is up | Go to Step 2 |
| STOPPED | App crashed or was stopped | Go to Step 1b |
| DEPLOYING | Mid-deployment | Wait 3 min, retry |
| FAILED | Deployment failed | Check deployment logs (Step 1c) |

**Step 1b — Read crash logs (app is STOPPED)**

```
Tool: get_server_logs
Source: mule_cloudhub_app
Window: 30 minutes
Level: ERROR, WARN
```

Look for these patterns:

```
Pattern: "OutOfMemoryError"
Meaning: Worker ran out of heap memory
Hypothesis: OOM crash → Fix R1 (restart) + Fix R2 (increase memory)

Pattern: "Could not initialise mule application"
Meaning: App config wrong — missing property or bad connector config
Hypothesis: Configuration error → check properties (Step 1c)

Pattern: "Unauthorized" or "401" or "403" in outbound calls
Meaning: Credentials for target system are wrong or expired
Hypothesis: Credential failure → Step 3 (credential investigation)

Pattern: "Connection timed out" or "Read timed out"
Meaning: Target system not responding within timeout window
Hypothesis: Timeout → Step 4 (timeout investigation)

Pattern: "Connection refused"
Meaning: Target system unreachable on that port
Hypothesis: Network/firewall → Step 5 (connectivity check)
```

**Step 1c — Check app properties (if misconfiguration suspected)**

```
Tool: run_command
Command ID: get_mule_app_properties
Input: { app_name: "{mule_app_name}", environment: "{environment}" }
```

Verify these exist and are non-empty (do not log values — only confirm presence):
- `http.host` / `http.port` — listener address
- `target.api.baseUrl` — outbound endpoint URL
- `target.api.clientId` — client credential (check exists, not value)
- `target.api.clientSecret` — client credential (check exists, not value)
- `timeout.connection` — connection timeout setting
- `timeout.response` — response timeout setting

---

### Step 2 — What is the error pattern in running app logs?

Run this even if app is STARTED — it may be running but failing on every request.

```
Tool: get_server_logs
Source: mule_cloudhub_app
Window: 30 minutes
Level: ERROR
```

**Pattern: 401 Unauthorized from target**

```
Example log line:
ERROR [mule.flow.order-api] HTTP response 401 Unauthorized
  from https://target-api.company.com/orders
  at connector: HTTP_Request_Configuration

→ Credential failure. The client ID or secret Mule is using
  is wrong, expired, or has been rotated by the target system team.
→ Go to Step 3: Credential Investigation
```

**Pattern: 403 Forbidden from target**

```
Example log line:
ERROR [mule.flow.order-api] HTTP response 403 Forbidden
  from https://target-api.company.com/orders

→ Credentials are valid but this Mule app's client ID does
  not have permission for this endpoint.
→ Go to Step 3: Credential Investigation (scope/permission issue)
```

**Pattern: Read timed out / Connection timed out**

```
Example log line:
ERROR [mule.flow.order-api] java.net.SocketTimeoutException: Read timed out
  after 30000ms
  connector: HTTP_Request_Configuration → target-api.company.com

→ Target system is reachable but not responding within the timeout window.
→ Go to Step 4: Timeout Investigation
```

**Pattern: Connection refused**

```
Example log line:
ERROR [mule.flow.order-api] java.net.ConnectException: Connection refused
  to target-api.company.com:8443

→ Target system port is not accepting connections.
   Either the service is down or firewall is blocking.
→ Go to Step 5: Connectivity Check
```

**Pattern: DataWeave transformation failed**

```
Example log line:
ERROR [mule.flow.order-api] Script 'transform_to_order_json' failed
  Cannot coerce Null to String at line 12

→ Input payload is missing a required field.
→ Go to Step 6: Data / Transform Investigation
```

---

### Step 3 — Credential Investigation (401/403 errors)

**Step 3a — Confirm which credential is failing**

```
Tool: get_server_logs
Source: mule_cloudhub_app
Window: 30 minutes
Filter: "401" OR "403" OR "Unauthorized" OR "Forbidden"
```

Extract: which endpoint URL is returning 401/403, which flow/connector.

**Step 3b — Check when the credential was last updated**

```
Tool: run_command
Command ID: check_secret_last_updated
Input: {
  secret_path: "{vault_path_for_credential}",
  secret_name: "target_api_client_secret"
}
```

If last updated > 90 days ago: likely expired rotation policy.
If last updated in last 48 hours: someone rotated the credential without updating Mule.

**Step 3c — Verify the credential exists in the secret store**

```
Tool: run_command
Command ID: check_secret_exists
Input: { secret_path: "{vault_path}" }
```

Confirms the secret key exists. Does NOT read or log the value.

**Step 3d — Check if target system team rotated credentials recently**

```
Tool: search_past_incidents
Query: "credential rotation {target_system_name} 401"
Window: 30 days
```

If a past incident mentions credential rotation: the target team likely rotated
again without notifying the integration team.

**After Step 3 — Raise communication to the right team:**

If credentials are confirmed wrong/expired:
- Do NOT attempt to guess or brute-force credentials
- Do NOT update credentials without getting the new values from the owning team
- Trigger: Communication Agent → notify target system team to provide new credentials
- Trigger: request_human_approval for any credential update action

---

### Step 4 — Timeout Investigation

A timeout means Mule can reach the target but the target is too slow.

**Step 4a — Is the timeout consistent or intermittent?**

```
Tool: get_server_logs
Source: mule_cloudhub_app
Window: 60 minutes
Filter: "timed out"
```

Count occurrences per 10-minute window:
- Consistent (every request): target system is likely down or degraded
- Intermittent (some requests succeed): target system is slow under load

**Step 4b — Check current timeout configuration**

```
Tool: run_command
Command ID: get_mule_app_properties
Input: { app_name: "{mule_app_name}", property: "timeout.*" }
```

Common Mule timeout properties:
- `timeout.connection` — time to establish TCP connection (default: 5000ms)
- `timeout.response` — time to wait for response after connected (default: 30000ms)
- `timeout.socket` — socket read timeout

If timeout values are very low (< 5000ms response timeout):
the target system may just be slower than the timeout allows.
This is a configuration issue, not a target system issue.

**Step 4c — Test target system response time directly**

```
Tool: run_command
Command ID: test_http_endpoint_with_timing
Input: {
  url: "{target_api_base_url}/health",
  method: "GET",
  timeout_ms: 60000,
  measure_response_time: true
}
```

If response time > current timeout setting:
→ Either increase timeout in Mule config OR investigate why target is slow.

If no response at all:
→ Target system is down → escalate to target system team.

**Step 4d — Check if the timeout started at a specific time (deployment correlation)**

```
Tool: search_past_incidents
Query: "{target_system_name} slow response degraded"
Window: 7 days
```

```
Tool: run_command
Command ID: get_deployment_history
Input: { service: "{target_system_name}", window_hours: 24 }
```

If target system was recently deployed → bad deployment causing slowness.

---

### Step 5 — Connectivity Check (connection refused)

```
Tool: run_command
Command ID: test_tcp_connectivity
Input: {
  from_host: "{mule_runtime_host}",
  to_host: "{target_host}",
  port: "{target_port}"
}
```

Expected: connection established in < 3 seconds.

If timeout: firewall is blocking → escalate to network team with:
- Source IP (Mule runtime IP / CloudHub NAT IP)
- Destination host and port
- Protocol (TCP/HTTPS)

```
Tool: run_command
Command ID: dns_lookup
Input: { hostname: "{target_host}", from: "{mule_runtime_host}" }
```

If DNS fails: wrong hostname in Mule config OR DNS issue.

---

### Step 6 — Data / Transform Investigation

```
Tool: get_server_logs
Source: mule_cloudhub_app
Window: 30 minutes
Filter: "DataWeave" OR "transform" OR "Cannot coerce" OR "Key not found"
```

Pull last 5 failed payloads from dead-letter queue (if configured):

```
Tool: run_command
Command ID: mule_get_dlq_messages
Input: { app_name: "{mule_app_name}", flow: "{failing_flow}", count: 5 }
```

Look for consistent null fields, wrong data types, or missing keys.
This is a development team issue — do not attempt to fix DataWeave transforms.
Escalate with: sample payload, exact error line, field that is failing.

---

## Resolution steps

### Fix R1 — Restart Mule application

**When:** App is STOPPED, workers are 0, root cause understood (OOM, deploy failure)
**Risk:** MEDIUM — ~3 minutes downtime for the integration
**Requires:** Human approval

```
Tool: run_command
Command ID: mule_restart_application
Input: { app_name: "{mule_app_name}", environment: "{environment}" }
```

Verify after: re-run Step 1, confirm status=STARTED, workers > 0.

---

### Fix R2 — Increase worker memory (OOM prevention)

**When:** OOM crash confirmed in logs, frequent recurrence
**Risk:** HIGH — requires redeployment, 5 minute downtime
**Requires:** Human approval

```
Tool: run_command
Command ID: mule_update_worker_config
Input: {
  app_name: "{mule_app_name}",
  environment: "{environment}",
  worker_size: "1GB",     ← increase from 0.5GB
  worker_count: 2
}
```

---

### Fix R3 — Update credentials in Mule (after team provides new values)

**When:** 401/403 confirmed, owning team has provided new credential values
**Risk:** HIGH — wrong credentials will keep the integration broken
**Requires:** Human approval + new credential value must come from secrets vault
**NEVER:** take credential value from ticket description or chat — always from vault

```
Tool: run_command
Command ID: mule_update_secret_reference
Input: {
  app_name: "{mule_app_name}",
  environment: "{environment}",
  secret_key: "target.api.clientSecret",
  vault_path: "{vault_path_for_new_secret}"
}
```

After update, app restarts automatically. Wait 3 minutes, then verify.

---

### Fix R4 — Increase timeout configuration

**When:** Timeouts confirmed, target system response time exceeds current timeout
**Risk:** LOW — only changes how long Mule waits, does not affect target
**Requires:** Human approval (config change)

```
Tool: run_command
Command ID: mule_update_property
Input: {
  app_name: "{mule_app_name}",
  environment: "{environment}",
  property_key: "timeout.response",
  property_value: "60000"    ← increase from 30000ms to 60000ms
}
```

---

### Fix R5 — Reprocess messages from dead-letter queue

**When:** Root cause fixed, messages that failed during the outage need reprocessing
**Risk:** MEDIUM — could create duplicate records if target system already processed some
**Requires:** Human approval + confirmation from target team that no duplicates exist

```
Tool: run_command
Command ID: mule_reprocess_dlq
Input: {
  app_name: "{mule_app_name}",
  flow: "{failing_flow}",
  message_ids: "{list_from_dlq_check}"
}
```

---

## Timeout handling — decision tree

```
Timeout detected in logs
        │
        ├── Is it consistent (every request)?
        │         │
        │         ├── Yes → Target system likely DOWN
        │         │         → Check target health endpoint (Step 4c)
        │         │         → If down: escalate to target team NOW
        │         │         → Do NOT increase Mule timeout (won't help if target is down)
        │         │
        │         └── No → Intermittent timeouts
        │                   → Check target response time (Step 4c)
        │                   → Compare with current timeout setting
        │                   │
        │                   ├── Response time > timeout: increase timeout (Fix R4)
        │                   └── Response time < timeout but still timing out:
        │                         → Load / concurrency issue on target
        │                         → Escalate to target team with response time data
        │
        ├── Did timeouts start at a specific time?
        │         │
        │         ├── After a deployment: suspect bad deploy on target system
        │         └── After a load spike: target system overwhelmed
        │
        └── What is the current timeout value?
                  │
                  ├── < 5000ms: almost certainly too low → Fix R4
                  ├── 5000-30000ms: normal range → investigate target system
                  └── > 60000ms: very high → target system is severely degraded
```

---

## Credential failure — communication workflow

When 401/403 is confirmed and credential rotation is the cause,
the agent NEVER updates credentials unilaterally.

The workflow is:

```
1. Confirm 401/403 error pattern in logs
2. Identify which target system and which credential is failing
3. Check secret_last_updated to understand rotation timeline
4. Search past incidents for recent rotation history
5. Spawn Communication Agent →
     Draft message to target system team:
       "Mule integration {app_name} is receiving 401 errors from your API
        since {first_error_time}. Our records show the last credential update
        was {last_updated}. Please provide updated credentials via the
        secure credential handoff process. Incident: {ticket_id}"
6. Set run state: AWAITING_CREDENTIAL_UPDATE
7. Wait for human to confirm new credential has been stored in vault
8. Request human approval to update Mule with vault reference
9. Execute Fix R3 with vault reference (never the raw value)
10. Verify integration restored
```

---

## Verification steps

After any fix, run in this order:

**V1 — App health (30 seconds)**
```
Tool: query_monitoring_api → anypoint_cloudhub
Confirm: status=STARTED, workers > 0, no ERROR logs in last 60 seconds
```

**V2 — End-to-end API call (2 minutes)**
```
Tool: run_command → mule_trigger_test_flow
Confirm: test payload flows through, target returns 200, no errors in logs
```

**V3 — No new errors (5 minutes)**
```
Tool: get_server_logs → filter ERROR
Confirm: zero new error entries after fix was applied
```

---

## Escalation paths

| Situation | Escalate to | What to include |
|---|---|---|
| Credentials wrong/expired | Target system team | App name, endpoint URL, first error time, ticket ID |
| Target system down | Target system team | Response from health check, timeout data |
| Firewall blocking | Network/Infra team | Source IP, destination host:port |
| DataWeave bug | Integration development team | Sample payload, exact error, flow name |
| CloudHub infrastructure issue | MuleSoft support | Org ID, environment, app name, region |
| Repeated OOM despite memory increase | Integration architecture team | Memory usage graph, payload size analysis |

---

## Do not

- Log or record credential values anywhere
- Attempt to guess or rotate credentials without the owning team
- Reprocess dead-letter queue without confirming no duplicates
- Increase timeouts above 120000ms without architecture review
- Restart the app more than twice without understanding the root cause
- Assume a 403 is the same as a 401 — 403 means auth worked but access denied (different fix)