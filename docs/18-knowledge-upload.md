# Knowledge Upload, KB & Graph Building
# docs/18-knowledge-upload.md

---

## The Three Knowledge Systems and Their Analogies

Before anything else — understand what each system is FOR.
Use these analogies when explaining to the team.

```
SYSTEM              ANALOGY                   WHAT IT ANSWERS
─────────────────────────────────────────────────────────────────────────────
Semantic KB         The company wiki          "How do I fix connection pool exhaustion?"
                    (Confluence, Notion)       Returns: step-by-step procedure

Episodic Memory     The team's memory         "Have we seen this before?"
(pgvector)          (the senior engineer       Returns: similar past incidents + what worked
                     who's been here 5 years)

Knowledge Graph     The org chart +           "Who owns this? What does it depend on?"
(Neo4j)             architecture diagram       Returns: structure, relationships, blast radius
                    on the wall
```

When someone asks "where should I put this?" use this decision tree:

```
Is it a PROCEDURE or DOCUMENT a human wrote?
  → Semantic KB (upload it)

Is it a PAST INCIDENT that was resolved?
  → Episodic Memory (system writes this automatically after each run)

Is it a FACT about a service, team, or system?
  → Knowledge Graph (seed it, then system grows it automatically)
```

---

## Part 1 — Semantic Knowledge Base Upload

### What belongs in the KB

```
✓ Upload these:
  Runbooks          "How to restart the payment service"
  SOPs              "Incident response procedure"
  Architecture docs "Payment service architecture overview"
  Known issues      "pgbouncer connection pool exhaustion — known issue + fix"
  FAQs              "Why does auth-service fail on Tuesdays?"
  Post-mortems      "Post-mortem: 2024-01-15 API outage"
  Config guides     "How to configure pgbouncer pool sizes"

✗ Do NOT upload these:
  Meeting notes (unstructured, low signal)
  Slack conversations
  Raw log files (go to episodic memory via runs)
  Code files (agents read from repo directly if needed)
  Spreadsheets with raw metrics (use monitoring API instead)
```

### Upload endpoint

```
POST /v1/knowledge/upload
Content-Type: multipart/form-data

Fields:
  file              — the document (required)
  source_type       — 'runbook'|'sop'|'architecture'|'known_issue'|'faq'|'post_mortem'
  title             — display title (required)
  service           — which service this applies to (optional but strongly recommended)
  category          — 'database'|'network'|'application'|'infrastructure'|'security'
  tags              — comma-separated tags
  severity_applicable — comma-separated: 'P1,P2' (which ticket priorities this helps)
  replace_doc_id    — if updating an existing document (replaces all its chunks)
```

```typescript
// POST /v1/knowledge/upload
// Response 201
{
  "doc_id":       "uuid",
  "title":        "pgbouncer Connection Pool Management",
  "source_type":  "runbook",
  "service":      "payment-service",
  "chunks_created": 6,
  "processing_status": "INDEXING",  // INDEXING → INDEXED → FAILED
  "status_url":   "/v1/knowledge/docs/uuid/status"
}

// GET /v1/knowledge/docs/:id/status
// Response 200
{
  "doc_id":     "uuid",
  "status":     "INDEXED",
  "chunks":     6,
  "indexed_at": "timestamp"
}
```

### Supported file formats

```
.md          Markdown — preferred format
.txt         Plain text
.pdf         PDF (text extracted, images ignored)
.docx        Word document (text extracted)
.html        HTML (tags stripped, content extracted)
.confluence  Confluence export format (if you export from Confluence)
.notion      Notion export format
```

### How documents are chunked (automatically)

The chunker reads the document type and applies the right strategy:

```typescript
// src/knowledge/chunker.ts

const CHUNKING_STRATEGIES: Record<SourceType, ChunkingStrategy> = {

  runbook: {
    // Chunk by H2/H3 heading — each procedure section is one chunk
    // A chunk = section heading + all content until next same-level heading
    // Min chunk: 100 tokens, Max chunk: 600 tokens
    // Overlap: 50 tokens (last paragraph of previous chunk prepended)
    strategy: 'heading_based',
    min_tokens: 100,
    max_tokens: 600,
    overlap_tokens: 50,
  },

  architecture: {
    // Chunk by component — each service/subsystem described is one chunk
    // Preserve code blocks intact (do not split across chunks)
    strategy: 'heading_based',
    min_tokens: 150,
    max_tokens: 500,
    overlap_tokens: 75,
  },

  known_issue: {
    // Each known issue = one chunk regardless of length
    // If length > 600 tokens, split at "Resolution" or "Fix" heading
    strategy: 'single_document',
    max_tokens: 800,
  },

  sop: {
    // Chunk by decision point/step group
    strategy: 'heading_based',
    min_tokens: 100,
    max_tokens: 500,
    overlap_tokens: 50,
  },

  faq: {
    // Each Q&A pair = one chunk
    strategy: 'qa_pairs',
    min_tokens: 50,
    max_tokens: 400,
  },

  post_mortem: {
    // Chunk into: summary, timeline, root cause, action items
    // These are natural sections in every post-mortem
    strategy: 'heading_based',
    min_tokens: 100,
    max_tokens: 700,
    overlap_tokens: 75,
  },
};
```

### Batch upload (for initial seeding)

For bulk uploads during initial setup, use the CLI tool:

```bash
# Install the CLI
npm install -g @ticket-agent/cli

# Configure
ticket-agent configure --api-key sk-... --url https://api.yourapp.com

# Upload a single file
ticket-agent kb upload \
  --file runbooks/pgbouncer-management.md \
  --source-type runbook \
  --service payment-service \
  --category database

# Upload an entire folder
ticket-agent kb upload-folder \
  --folder ./runbooks \
  --source-type runbook \
  --category database

# Upload with metadata from a YAML manifest
ticket-agent kb upload-manifest --manifest ./kb-manifest.yaml
```

Manifest format for bulk upload:

```yaml
# kb-manifest.yaml
uploads:
  - file: runbooks/pgbouncer-management.md
    source_type: runbook
    title: "pgbouncer Connection Pool Management"
    service: payment-service
    category: database
    tags: [postgresql, pgbouncer, connections]
    severity_applicable: [P1, P2]

  - file: runbooks/nginx-reload.md
    source_type: runbook
    title: "Nginx Configuration Reload"
    service: api-gateway
    category: network
    tags: [nginx, reload, config]
    severity_applicable: [P1, P2, P3]

  - file: known-issues/tuesday-auth-failure.md
    source_type: known_issue
    title: "Auth Service Failure Pattern — Tuesday Mornings"
    service: auth-service
    category: application
    tags: [auth, recurring, known-issue]
    severity_applicable: [P1]

  - folder: architecture/
    source_type: architecture
    category: infrastructure
    # each file in the folder gets its own doc
    file_to_service_map:
      payment-arch.md: payment-service
      auth-arch.md: auth-service
      billing-arch.md: billing-api
```

---

## Part 2 — Knowledge Graph Seeding

The graph grows automatically after every resolved incident.
But you need to seed it with the base structure first.
Without seeding, the graph starts empty and agents have no structural context.

### What you need to seed

```
Priority 1 — seed these before first run:
  Services          All your microservices/APIs
  Teams             Who owns what
  Dependencies      What calls what (the call graph)

Priority 2 — seed these in the first week:
  Known failure modes   Common problems you already know about
  Persons               On-call engineers

Priority 3 — let the system build these automatically:
  Incidents             Written after every resolved run
  Remediation actions   Written after every resolved run
  Alert gaps            Flagged during runs
```

### Seed via CSV upload (easiest for initial setup)

```
POST /v1/graph/seed
Content-Type: multipart/form-data

Fields:
  file        — CSV file
  entity_type — 'services'|'teams'|'dependencies'|'persons'|'failure_modes'
```

### Services CSV format

```csv
name,tier,sla_tier,repo_url,team_name,environment
payment-service,tier-1,gold,https://github.com/org/payment-service,Payments Team,production
auth-service,tier-1,gold,https://github.com/org/auth-service,Identity Team,production
billing-api,tier-2,silver,https://github.com/org/billing-api,Billing Team,production
user-api,tier-1,gold,https://github.com/org/user-api,Core Platform,production
order-service,tier-2,silver,https://github.com/org/order-service,Commerce Team,production
db-proxy,tier-1,gold,,Platform Team,production
redis-cache,tier-1,gold,,Platform Team,production
```

### Teams CSV format

```csv
name,slack_channel,oncall_schedule_id
Payments Team,#payments-oncall,PD-SCHEDULE-001
Identity Team,#identity-oncall,PD-SCHEDULE-002
Billing Team,#billing-oncall,PD-SCHEDULE-003
Core Platform,#platform-oncall,PD-SCHEDULE-004
Commerce Team,#commerce-oncall,PD-SCHEDULE-005
Platform Team,#platform-oncall,PD-SCHEDULE-004
```

### Dependencies CSV format

```csv
from_service,to_service,criticality,dependency_type
payment-service,db-proxy,hard,sync
payment-service,redis-cache,hard,sync
payment-service,auth-service,hard,sync
billing-api,payment-service,hard,sync
user-api,auth-service,hard,sync
user-api,db-proxy,hard,sync
order-service,payment-service,hard,sync
order-service,user-api,hard,sync
db-proxy,postgres-primary,hard,sync
```

`criticality` values:
- `hard` — the calling service fails if this dependency is down
- `soft` — the calling service degrades but does not fail

`dependency_type` values:
- `sync` — synchronous HTTP/gRPC call
- `async` — publishes to a queue, consumed asynchronously
- `data` — reads data from (e.g. reads from a shared database)

### Known failure modes CSV format

```csv
name,description,category,avg_resolution_mins,service
connection_pool_exhaustion,PostgreSQL or pgbouncer connection pool hits maximum connections,database,23,payment-service
redis_memory_pressure,Redis memory usage above 90% — eviction causing cache misses,cache,15,redis-cache
auth_token_expiry_cascade,Auth service token refresh storm after deployment,authentication,12,auth-service
slow_query_cascade,Long-running database queries blocking connection slots,database,35,db-proxy
nginx_worker_crash,Nginx worker processes dying — gateway returning 502s,network,8,api-gateway
```

### Persons CSV format

```csv
name,email,role,team_name,slack_user_id
Alice Chen,alice@company.com,Staff Engineer,Payments Team,U01234
Bob Smith,bob@company.com,SRE Lead,Platform Team,U05678
Carol Wu,carol@company.com,Senior Engineer,Identity Team,U09012
```

### Seed via API (programmatic)

For teams that want to automate seeding from their existing CMDB or service registry:

```typescript
// POST /v1/graph/seed/service
{
  "name":          "payment-service",
  "tier":          "tier-1",
  "sla_tier":      "gold",
  "repo_url":      "https://github.com/org/payment-service",
  "team_name":     "Payments Team",
  "environment":   "production"
}

// POST /v1/graph/seed/dependency
{
  "from_service":      "payment-service",
  "to_service":        "db-proxy",
  "criticality":       "hard",
  "dependency_type":   "sync"
}

// POST /v1/graph/seed/failure_mode
{
  "name":                   "connection_pool_exhaustion",
  "description":            "string",
  "category":               "database",
  "avg_resolution_mins":    23,
  "service":                "payment-service"
}
```

### Seeding from your existing tools

If you already have service topology documented somewhere:

```typescript
// Seed from Backstage (service catalog)
ticket-agent graph seed --from backstage \
  --backstage-url https://backstage.yourcompany.com \
  --backstage-token $BACKSTAGE_TOKEN

// Seed from Datadog service map
ticket-agent graph seed --from datadog \
  --datadog-api-key $DD_API_KEY \
  --datadog-app-key $DD_APP_KEY

// Seed from AWS service discovery
ticket-agent graph seed --from aws-service-discovery \
  --aws-region us-east-1 \
  --namespace your-namespace

// Seed from Kubernetes (reads service definitions)
ticket-agent graph seed --from kubernetes \
  --kubeconfig ~/.kube/config \
  --namespace production
```

---

## Part 3 — How the Graph Grows Automatically

After every resolved run, the Knowledge Creator agent generates `graph_updates.json`.
A nightly batch job reads these files and applies them to Neo4j.

### What grows automatically (zero human effort after initial seed)

```
After each resolved incident, the system automatically:

1. Creates a new Incident node
   (:Incident {id, ticket_id, root_cause, resolution_mins, ...})

2. Links incident to affected service
   (:Incident)-[:AFFECTED]->(:Service)

3. Links incident to failure mode (creates FailureMode node if new)
   (:Incident)-[:CAUSED_BY]->(:FailureMode)
   Updates: FailureMode.count++, FailureMode.avg_resolution_mins (rolling avg)

4. Links incident to remediation action (creates if new)
   (:Incident)-[:RESOLVED_BY]->(:RemediationAction)
   Updates: RemediationAction.success_rate (rolling avg)

5. Links remediation to KB document (if a KB doc was used)
   (:RemediationAction)-[:DOCUMENTED_IN]->(:KBDocument)

6. Flags missing alerts
   (:Service)-[:MISSING_ALERT]->(:AlertGap)
   Created when agent detects system was in bad state before alert fired
```

### After 50 resolved incidents, what the graph knows automatically

```
payment-service has had:
  14 connection_pool_exhaustion incidents (avg 23 min each)
  6  slow_query_cascade incidents (avg 35 min each)
  3  redis_cache_miss_storm incidents (avg 12 min each)

connection_pool_exhaustion is most reliably fixed by:
  pgbouncer_reload (success_rate: 0.87, 12 uses)
  increase_max_client_conn (success_rate: 0.92, 5 uses)

payment-service is depended on by:
  billing-api, order-service (blast radius when payment-service is down)

Missing alerts flagged:
  payment-service: no alert for pg_stat_activity count > 80
  auth-service: no alert for token refresh rate > 1000/min
```

---

## Part 4 — Reviewing and Validating Knowledge

### KB quality dashboard

```
GET /v1/admin/knowledge/quality
Response:
{
  "kb": {
    "total_docs": 48,
    "total_chunks": 312,
    "avg_confidence": 0.81,
    "stale_chunks": 14,        // last_validated > 90 days ago
    "low_confidence_chunks": 8  // confidence < 0.60
  },
  "gaps": {
    "total_open": 7,
    "high_priority": 3,
    "pending_review": 5
  },
  "graph": {
    "services": 24,
    "teams": 8,
    "failure_modes": 18,
    "incidents_recorded": 142,
    "missing_alerts": 6
  }
}
```

### Validating a KB document

When your team updates a runbook, mark it re-validated:

```
POST /v1/knowledge/docs/:id/validate
{
  "validated_by": "alice@company.com",
  "notes": "Tested procedure on staging — steps 3-5 updated"
}
```

This sets `last_validated = NOW()` and bumps `confidence` back to 0.9.

### Stale chunk alert

A weekly job emails document owners when chunks fall below confidence 0.6:

```
Subject: [Ticket Agent] 3 knowledge base documents need review

The following documents have not been validated in 90+ days and may be outdated:

1. pgbouncer Connection Pool Management
   Last validated: 2024-10-15 by Bob Smith
   Times retrieved in past 30 days: 18
   Times it helped resolve a ticket: 12
   → Review: https://admin.yourapp.com/knowledge/docs/abc123

2. Nginx Reload Procedure
   Last validated: 2024-09-01 by Carol Wu
   Times retrieved: 6  Times helpful: 3
   → Review: https://admin.yourapp.com/knowledge/docs/def456
```

---

## Part 5 — Admin Panel Knowledge View (wire to the UI)

The admin panel (see docs/19-admin-panel.md) shows:

```
Knowledge Base tab:
  - Table of all documents: title, source_type, service, confidence,
    last_validated, times_retrieved_30d, times_helpful_30d
  - Filter by: service, category, status (stale/healthy/low-confidence)
  - Actions: Upload new, Re-validate, Replace, Delete
  - Click document → see all its chunks with individual confidence scores

Gaps tab:
  - Table of open gap reports: description, service, priority, estimated_steps_saved
  - Filter by: status (PENDING/APPROVED/REJECTED), priority
  - Click gap → see full gap report + draft (KB or skill)
  - Actions: Approve draft → publishes to KB, Reject, Assign to person

Graph tab:
  - Node counts: services, teams, failure_modes, incidents
  - Top failure modes by frequency (bar chart)
  - Services with most incidents (table)
  - Missing alert gaps (table)
  - Click service → show its dependencies, known failures, recent incidents
```
