---
name: mule-sap-api-connectivity
description:
  Diagnoses MuleSoft Anypoint Platform connectivity failures involving SAP
  ECC and non-SAP systems. Covers IDoc delivery, BAPI calls, JCo connections,
  RFC destinations, and Mule flow failures in CloudHub and on-premise runtimes.
license: MIT
metadata:
  author: ops-agent
  version: '1.0.0'
---

## When to load this skill

Load when any of these match:

**Symptoms in ticket:**
- "Mule API down" / "Mule flow failing" / "Mule connector error"
- "SAP not receiving data from Mule" / "Mule not sending to SAP"
- "API connectivity issue between SAP and [any system]"
- "IDoc not delivered" / "BAPI call failed from Mule"
- "JCo connection refused" / "RFC destination unreachable"
- "Mule CloudHub app stopped" / "Mule worker unresponsive"
- "Integration broken" + SAP ECC mentioned anywhere
- HTTP 5xx errors from a Mule-managed API

**Affected services:**
- Any service tagged `mule` or `anypoint` or `integration`
- SAP ECC + any middleware layer

**Categories:**
- integration, middleware, api-connectivity, sap, mulesoft

---

## Architecture context — read this first

Before running any diagnostic, understand the topology.
A Mule API connectivity failure can break at any of six layers.

```
[Source System]
  e.g. Salesforce, Non-SAP DB, REST client, SAP-adjacent app
        │
        ▼
[MuleSoft Anypoint Platform]
  CloudHub (managed) or On-Premise Runtime
  ┌─────────────────────────────────────────────┐
  │  Mule Application (flow)                    │
  │    ├── Inbound: HTTP Listener / SFTP / JMS  │
  │    ├── Transform (DataWeave)                │
  │    └── Outbound: SAP Connector / HTTP / DB  │
  └─────────────────────────────────────────────┘
        │
        ▼ (SAP path)
[SAP JCo Library]  ←→  [RFC Destination in SM59]
        │
        ▼
[SAP ECC Application Server]
  ├── ABAP stack
  ├── BAPI / Function Module
  └── IDoc processing (WE02, WE05)
        │
        ▼
[SAP Database layer]  ←  (rarely the issue but worth knowing)
```

**The failure is ALWAYS in one of these layers:**
1. Mule app itself (crashed, misconfigured, OOM)
2. Mule → SAP network path (firewall, VPN, port)
3. SAP JCo connector config (wrong credentials, wrong host)
4. SAP RFC destination (SM59 misconfigured or SAP down)
5. SAP ABAP layer (BAPI error, authorisation, lock)
6. Source system sending bad data (DataWeave transform fails)

Your job is to identify which layer as fast as possible.

---

## Diagnostic sequence

Work top-down. Confirm each layer is healthy before going deeper.

---

### Layer 1 — Is the Mule application running?

**Check 1a — CloudHub application status (if on CloudHub)**

```
Tool: query_monitoring_api
API: anypoint_cloudhub
Query: get_application_status
Params: { app_name: "{mule_app_name}", environment: "{environment}" }
```

Expected: `status: "STARTED"`, `workers: N` (N > 0)

**Interpret:**
- `STOPPED` → app crashed or was manually stopped → go to Check 1b
- `DEPLOYING` → deployment in progress → wait 3 minutes, re-check
- `FAILED` → deployment failed → check deployment logs (Check 1c)
- `STARTED` but workers = 0 → OOM or worker crash → check worker logs

**Check 1b — Recent application logs (last 30 minutes)**

```
Tool: get_server_logs
Log source: mule_cloudhub_app
Time window: 30 minutes
Filter level: ERROR, WARN
```

Look for:
- `OutOfMemoryError` → worker ran out of heap → worker restarted, scale up or increase heap
- `ConnectException: Connection refused` → cannot reach SAP → go to Layer 3
- `com.sap.conn.jco.JCoException` → JCo error → go to Layer 3
- `Could not initialise mule application` → bad config/deployment → go to Check 1c
- `DataWeave transformation failed` → bad input data → go to Layer 6
- `Timeout waiting for connection from pool` → connection pool exhausted → check concurrency

**Check 1c — Deployment logs (if app failed to start)**

```
Tool: get_server_logs
Log source: mule_deployment_log
Time window: 60 minutes
Filter: ERROR
```

Look for:
- `Property not found` → missing environment variable or config property
- `Could not load connector` → connector dependency missing
- `Port already in use` → port conflict on on-premise runtime
- `ClassNotFoundException` → JAR dependency missing in application

**On-premise Mule runtime (if not CloudHub):**

```
Tool: run_command
Command ID: check_mule_service_status
Params: { host: "{mule_runtime_host}" }
```
Checks: `systemctl status mule` or equivalent

---

### Layer 2 — Can Mule reach SAP on the network?

Run this BEFORE checking SAP config — no point diagnosing JCo if the port is blocked.

**Check 2a — TCP connectivity to SAP application server**

```
Tool: run_command
Command ID: test_tcp_connectivity
Params: {
  from_host: "{mule_runtime_host}",
  to_host: "{sap_app_server_host}",
  port: "{sap_gateway_port}"   // typically 3300 + system_number e.g. 3300 for sysnr 00
}
```

SAP ports to check:
- RFC/JCo: `33{system_number}` — e.g. 3300 for sysnr 00, 3301 for sysnr 01
- HTTP/HTTPS (if SAP Gateway/OData): 8000, 44300, or custom
- SAP Message Server: `36{system_number}`

Expected: connection established within 3 seconds
If timeout: firewall is blocking → escalate to network team with specific port + source IP

**Check 2b — DNS resolution**

```
Tool: run_command
Command ID: dns_lookup
Params: { host: "{sap_app_server_host}", from: "{mule_runtime_host}" }
```

Expected: resolves to SAP app server IP
If fails: DNS issue or wrong hostname in Mule config

---

### Layer 3 — Is the JCo connector / SAP connection configured correctly?

**Check 3a — JCo connection test from Mule**

```
Tool: run_command
Command ID: mule_jco_connection_test
Params: {
  app_name: "{mule_app_name}",
  connector_config_name: "{jco_config_name}"
}
```

This calls the Mule management API to test the SAP connector connection.
Expected: `{"status": "OK", "serverInfo": "SAP ECC 6.0 / {sid}"}`

Common JCo errors and meaning:

| Error | Meaning | Fix |
|---|---|---|
| `JCO_ERROR_COMMUNICATION` | Cannot reach SAP — network or port | Check Layer 2, verify SM59 |
| `JCO_ERROR_LOGON_FAILURE` | Wrong credentials | Check user/password in Mule connector config |
| `JCO_ERROR_DESTINATION` | RFC destination not found | Verify RFC destination name in Mule matches SM59 |
| `JCO_ERROR_SYSTEM_FAILURE` | SAP ABAP error | Check SAP side — go to Layer 4 |
| `JCO_ERROR_RESOURCE` | Connection pool full | Too many concurrent requests — check concurrency limits |
| `Client '000' not allowed` | Wrong SAP client number | Fix client in Mule connector config |
| `ICM_HTTP_CONNECTION_FAILED` | SAP ICM not running | SAP basis team — ICM service restart |

**Check 3b — Mule connector configuration values**

```
Tool: run_command
Command ID: get_mule_app_properties
Params: { app_name: "{mule_app_name}", environment: "{environment}" }
```

Verify these fields are correct (do NOT log values — just confirm they exist and are non-empty):
- `sap.jco.ashost` — SAP application server hostname
- `sap.jco.sysnr` — SAP system number (2 digits: 00, 01 etc)
- `sap.jco.client` — SAP client (e.g. 100, 200, 300)
- `sap.jco.user` — RFC user in SAP
- `sap.jco.lang` — language (EN)
- `sap.jco.pool.capacity` — connection pool size
- `sap.jco.peak.limit` — max connections

---

### Layer 4 — Is SAP ECC healthy and accepting RFC calls?

**Check 4a — SAP system availability (SM50 / SM66)**

```
Tool: run_command
Command ID: sap_sm50_workprocess_status
Params: { sap_host: "{sap_app_server_host}", sid: "{sap_sid}" }
```

Look for:
- All work processes `WAITING` or `RUNNING` — healthy
- Many processes in `PRIV` (private mode) — memory pressure on SAP
- Processes stuck in `Running` for > 10 minutes — locked or long-running transaction
- 0 free dialog work processes — SAP is overloaded

**Check 4b — RFC destination test from SAP side (SM59)**

This tests whether SAP can route RFC calls properly.

```
Tool: run_command
Command ID: sap_sm59_connection_test
Params: {
  sap_host: "{sap_app_server_host}",
  rfc_destination: "{rfc_destination_name}"
}
```

Expected: Connection test successful
If fails: RFC destination misconfigured in SAP — SAP basis team needs to fix SM59

**Check 4c — SAP user authorisation (SU53)**

If JCo connects but BAPI calls fail with authorisation errors:

```
Tool: run_command
Command ID: sap_check_user_auth
Params: {
  sap_host: "{sap_app_server_host}",
  username: "{mule_rfc_user}",
  bapi_name: "{failing_bapi_name}"
}
```

Look for: missing authorisation objects — if found, SAP BASIS needs to add auth to the RFC user

**Check 4d — IDoc status (if issue is IDoc-based)**

If Mule sends IDocs to SAP and they are not being processed:

```
Tool: run_command
Command ID: sap_we02_idoc_status
Params: {
  sap_host: "{sap_app_server_host}",
  status_filter: "error",
  time_window_mins: 60
}
```

IDoc status codes to know:
- `02` — Dispatched — sent but not yet processed — check inbound queue
- `04` — Error in control — IDoc structure wrong — DataWeave transform issue
- `25` — Processing despite syntax errors — SAP accepted but has issues
- `51` — Error: application document not posted — BAPI failed — check IDoc error message
- `53` — Application document posted — success

If IDocs are stuck in status `02` for > 5 minutes:
```
Tool: run_command
Command ID: sap_bd87_reprocess_idocs
Params: { sap_host: "{sap_app_server_host}", status: "02", time_window_mins: 60 }
```
Requires MEDIUM risk approval.

---

### Layer 5 — Non-SAP side (source or target system)

If the issue is Mule connecting TO a non-SAP system (REST API, database, SFTP, Salesforce):

**Check 5a — HTTP endpoint reachability**

```
Tool: run_command
Command ID: test_http_endpoint
Params: {
  from_host: "{mule_runtime_host}",
  url: "{target_endpoint_url}",
  method: "GET"
}
```

Expected: HTTP 200 or 401 (reachable but needs auth)
If 5xx or timeout: target system is down — escalate to that system's team

**Check 5b — Database connectivity (if target is DB)**

```
Tool: run_command
Command ID: test_db_connectivity
Params: {
  from_host: "{mule_runtime_host}",
  db_host: "{db_host}",
  db_port: "{db_port}",
  db_type: "{oracle|mssql|postgres}"
}
```

**Check 5c — Salesforce / external SaaS connectivity**

```
Tool: query_monitoring_api
API: anypoint_platform
Query: get_connector_status
Params: { connector_type: "salesforce", app_name: "{mule_app_name}" }
```

---

### Layer 6 — DataWeave transform / bad input data

If the Mule app is running and connections are fine but flows are failing:

**Check 6a — Recent DataWeave errors**

```
Tool: get_server_logs
Log source: mule_cloudhub_app
Time window: 30 minutes
Filter: DataWeave
```

Look for:
- `Cannot coerce String to Number` → source data type mismatch
- `Expected String but got Null` → required field is null in source
- `Key not found` → source data missing expected field
- `Invalid date format` → date format mismatch between systems

**Check 6b — Sample the failing payload**

```
Tool: run_command
Command ID: mule_get_error_payload_sample
Params: { app_name: "{mule_app_name}", flow_name: "{failing_flow}", count: 5 }
```

This pulls the last 5 failed message payloads from the Mule error queue (if dead-letter queue is configured).
Look for: pattern in what fields are null, wrong, or missing.

---

## Resolution steps

### Fix R1 — Restart Mule application (most common fix)

**When to use:** App is STOPPED or workers crashed, no config change needed

```
Tool: run_command
Command ID: mule_restart_application
Params: { app_name: "{mule_app_name}", environment: "{environment}" }
Requires: MEDIUM risk approval
Expected: App returns to STARTED within 3 minutes
```

Verify after: re-run Check 1a — confirm status = STARTED and all workers healthy

### Fix R2 — Update Mule connector credentials

**When to use:** JCo_ERROR_LOGON_FAILURE — password expired or wrong credentials

```
Tool: run_command
Command ID: mule_update_property
Params: {
  app_name: "{mule_app_name}",
  environment: "{environment}",
  property_key: "sap.jco.password",
  property_value: "{new_password_from_vault}"
}
Requires: HIGH risk approval
```

Note: password must be fetched from secrets vault — never from ticket content or agent memory.
After update, app restarts automatically. Wait 3 minutes then run Check 3a.

### Fix R3 — Increase Mule worker memory (OOM fix)

**When to use:** Workers crashed with OutOfMemoryError repeatedly

```
Tool: run_command
Command ID: mule_update_worker_config
Params: {
  app_name: "{mule_app_name}",
  environment: "{environment}",
  worker_size: "1GB",        // increase from current: 0.5GB → 1GB → 2GB → 4GB
  worker_count: 2
}
Requires: HIGH risk approval
Expected: App redeploys with new memory config — takes 5 minutes
```

### Fix R4 — Reprocess failed IDocs in SAP

**When to use:** IDocs stuck in error status 51 with known-good fix

```
Tool: run_command
Command ID: sap_bd87_reprocess_idocs
Params: {
  sap_host: "{sap_app_server_host}",
  idoc_numbers: "{comma_separated_idoc_numbers}",  // from Check 4d
  reprocess_mode: "individual"
}
Requires: MEDIUM risk approval
```

### Fix R5 — Clear Mule connection pool (pool exhaustion)

**When to use:** JCO_ERROR_RESOURCE — connection pool full

```
Tool: run_command
Command ID: mule_restart_connector_pool
Params: { app_name: "{mule_app_name}", connector_name: "{jco_connector_name}" }
Requires: LOW risk approval
Expected: Pool clears within 30 seconds, existing connections not disrupted
```

### Fix R6 — Trigger manual flow rerun (if data was not processed)

**When to use:** Flow failed on specific messages, root cause fixed, messages need reprocessing

```
Tool: run_command
Command ID: mule_rerun_failed_flow
Params: {
  app_name: "{mule_app_name}",
  flow_name: "{flow_name}",
  message_ids: "{ids_from_dead_letter_queue}"
}
Requires: MEDIUM risk approval
```

---

## Verification steps

After any fix, always verify at these three levels:

**Level 1 — Application health (30 seconds after fix)**
```
Tool: run_command
Command ID: mule_restart_application → Check 1a
Expected: status=STARTED, all workers UP
```

**Level 2 — End-to-end connectivity test (1 minute after fix)**
```
Tool: run_command
Command ID: mule_jco_connection_test → Check 3a
Expected: JCo connection OK, SAP server info returned
```

**Level 3 — Flow execution test (2 minutes after fix)**
```
Tool: run_command
Command ID: mule_trigger_test_flow
Params: { app_name: "{mule_app_name}", flow_name: "{health_check_flow}" }
Expected: Test message flows from Mule to SAP and returns success
```

**Level 4 — Check no new errors (5 minutes after fix)**
```
Tool: get_server_logs
Log source: mule_cloudhub_app
Time window: 5 minutes
Filter: ERROR
Expected: Zero new ERROR entries
```

---

## Escalation paths

| Situation | Escalate to | Information to include |
|---|---|---|
| SAP work processes full / SAP unresponsive | SAP BASIS team | SM50 screenshot, affected SID, time of issue |
| SAP authorisation missing | SAP BASIS + Security | Username, BAPI name, missing auth objects from SU53 |
| Firewall blocking port | Network / Infrastructure team | Source IP, destination IP, port number |
| SM59 RFC destination broken | SAP BASIS | Destination name, error from connection test |
| CloudHub infrastructure issue (not app) | MuleSoft Support | CloudHub region, org ID, environment, app name |
| Non-SAP target system down | Owning team for that system | Endpoint URL, error response, time of failure |
| DataWeave transform logic wrong | Integration development team | Sample failing payload, DataWeave error message, flow name |

---

## Do not do these

- Do not restart the Mule app more than twice without understanding the root cause — if it crashes twice, the problem is not the restart
- Do not change SAP user passwords without coordinating with SAP BASIS — the same user may be used by multiple integrations
- Do not reprocess IDocs in bulk without understanding why they failed — reprocessing bad data into SAP can create duplicate documents
- Do not increase worker count without increasing memory — adding workers with the same OOM pressure just creates more OOM crashes
- Do not modify SM59 RFC destinations directly — this is SAP BASIS responsibility
- Do not expose JCo credentials in logs or working memory — always reference secrets vault

---

## Key terminology for SAP ECC + Mule

| Term | Meaning |
|---|---|
| JCo | SAP Java Connector — the library Mule uses to talk to SAP RFC |
| RFC | Remote Function Call — SAP's remote procedure protocol |
| BAPI | Business Application Programming Interface — SAP standard function modules |
| IDoc | Intermediate Document — SAP's format for async data exchange |
| SM59 | SAP transaction to manage RFC destinations |
| WE02/WE05 | SAP transactions to view IDoc status |
| BD87 | SAP transaction to reprocess failed IDocs |
| SU53 | SAP transaction to check what authorisation a user is missing |
| SM50 | SAP transaction for work process overview |
| SID | SAP System ID — e.g. PRD (production), QAS (QA), DEV (development) |
| Client | SAP logical unit within a system — e.g. 100, 200, 300 |
| sysnr | SAP system number — 2 digits — determines RFC port (33{sysnr}) |
| DataWeave | MuleSoft's transformation language for mapping data between formats |
| CloudHub | MuleSoft's managed cloud runtime for Mule applications |
| Anypoint | MuleSoft's integration platform (CloudHub + API Manager + Exchange) |