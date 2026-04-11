# ServiceNow Integration — Ticket Agent System
# docs/17-servicenow.md

---

## How It Works End to End

User gives a ticket number (e.g. INC0012345).
The API fetches the full ticket from ServiceNow.
The system normalises it into the internal ticket format.
A run is created and agents start working.
As the agent makes progress, it writes updates back to ServiceNow.
When resolved, it closes the ServiceNow incident with a full resolution note.

The user never needs to copy-paste ticket content.
The ticket number is the only input.

---

## 1. Two Ways to Trigger

### Manual trigger (user submits ticket number)

```
POST /v1/tickets
{
  "ticket_number": "INC0012345",
  "source": "servicenow"
}
```

The API calls ServiceNow, fetches the full incident, normalises it, creates the run.

### Automatic trigger (ServiceNow webhook)

ServiceNow sends a webhook when an incident is created or escalated.
No human needed — the system starts automatically on P1/P2 incidents.

```
POST /webhooks/tickets/servicenow
{
  "sys_id": "abc123def456",
  "number": "INC0012345",
  "state": "1",
  "priority": "1",
  ... (raw ServiceNow payload)
}
```

---

## 2. ServiceNow API Client

```typescript
// src/integrations/servicenow/client.ts

export class ServiceNowClient {
  private baseUrl: string;    // https://yourcompany.service-now.com
  private auth: ServiceNowAuth;

  // Fetch full incident by ticket number
  async getIncident(ticketNumber: string): Promise<ServiceNowIncident> {
    const response = await this.get('/api/now/table/incident', {
      params: {
        sysparm_query: `number=${ticketNumber}`,
        sysparm_fields: [
          'sys_id', 'number', 'short_description', 'description',
          'priority', 'urgency', 'impact', 'state', 'category',
          'subcategory', 'assignment_group', 'assigned_to',
          'cmdb_ci',                    // affected CI (configuration item)
          'business_service',           // affected business service
          'u_affected_services',        // custom field: comma-separated services
          'u_error_rate',               // custom field
          'u_latency_p99',              // custom field
          'u_alert_source',             // custom field: 'datadog'|'pagerduty' etc
          'work_notes',                 // internal notes
          'comments',                   // customer-visible comments
          'opened_by', 'caller_id',
          'opened_at', 'sys_updated_on',
          'close_code', 'close_notes',
          'correlation_id',             // links to monitoring alert
          'parent_incident',
          'child_incidents',
          'attachments'                 // file attachments
        ].join(','),
        sysparm_limit: 1,
        sysparm_display_value: 'all'   // get both value and display_value
      }
    });

    if (!response.result || response.result.length === 0) {
      throw new NotFoundError(`ServiceNow incident ${ticketNumber} not found`);
    }

    return response.result[0];
  }

  // Fetch attachment content
  async getAttachment(sysId: string): Promise<Buffer> {
    return this.getBlob(`/api/now/attachment/${sysId}/file`);
  }

  // Add work note (internal, not visible to customer)
  async addWorkNote(sysId: string, note: string): Promise<void> {
    await this.patch(`/api/now/table/incident/${sysId}`, {
      work_notes: note
    });
  }

  // Add comment (customer-visible)
  async addComment(sysId: string, comment: string): Promise<void> {
    await this.patch(`/api/now/table/incident/${sysId}`, {
      comments: comment
    });
  }

  // Update incident fields
  async updateIncident(sysId: string, fields: Partial<ServiceNowUpdateFields>): Promise<void> {
    await this.patch(`/api/now/table/incident/${sysId}`, fields);
  }

  // Resolve incident
  async resolveIncident(sysId: string, resolution: ServiceNowResolution): Promise<void> {
    await this.patch(`/api/now/table/incident/${sysId}`, {
      state: '6',                       // 6 = Resolved in ServiceNow
      close_code: resolution.closeCode, // 'Solved (Permanently)' etc
      close_notes: resolution.notes,
      u_root_cause: resolution.rootCause,
      u_resolution_action: resolution.action,
      u_agent_run_id: resolution.runId  // custom field — links back to our system
    });
  }

  // Fetch CI (Configuration Item) details — gives us server/service metadata
  async getCi(ciSysId: string): Promise<ServiceNowCI> {
    return this.get(`/api/now/table/cmdb_ci_service/${ciSysId}`, {
      params: {
        sysparm_fields: 'name,sys_class_name,operational_status,u_team,u_environment'
      }
    });
  }
}
```

---

## 3. Priority and State Mapping

```typescript
// src/integrations/servicenow/mapping.ts

// ServiceNow priority → internal priority
export const PRIORITY_MAP: Record<string, TicketPriority> = {
  '1': 'P1',   // Critical
  '2': 'P2',   // High
  '3': 'P3',   // Moderate
  '4': 'P4',   // Low
  '5': 'P4',   // Planning
};

// ServiceNow state → human readable
export const STATE_MAP: Record<string, string> = {
  '1': 'New',
  '2': 'In Progress',
  '3': 'On Hold',
  '6': 'Resolved',
  '7': 'Closed',
  '8': 'Cancelled',
};

// ServiceNow category → internal category hint for triage
export const CATEGORY_MAP: Record<string, string> = {
  'database':    'database',
  'network':     'network',
  'application': 'application',
  'hardware':    'infrastructure',
  'software':    'application',
  'security':    'security',
  'inquiry':     'general',
};
```

---

## 4. Normaliser — ServiceNow → Internal Format

```typescript
// src/integrations/servicenow/normaliser.ts

export async function normaliseIncident(
  raw: ServiceNowIncident,
  client: ServiceNowClient,
  tenantId: string
): Promise<NormalisedTicket> {

  // Fetch CI for richer service context
  let ciData: ServiceNowCI | null = null;
  if (raw.cmdb_ci?.value) {
    ciData = await client.getCi(raw.cmdb_ci.value).catch(() => null);
  }

  // Parse affected services
  // Sources in priority order:
  // 1. Custom field u_affected_services (comma-separated)
  // 2. CMDB CI name
  // 3. Business service name
  // 4. Extract from description using regex
  const affectedServices = parseAffectedServices(raw, ciData);

  // Parse error rate and latency from custom fields or description
  const errorRate = parseFloat(raw.u_error_rate?.value || '0') || extractErrorRate(raw.description?.value);
  const latencyP99 = parseFloat(raw.u_latency_p99?.value || '0') || extractLatency(raw.description?.value);

  // Fetch attachments (logs, screenshots)
  const attachments = await fetchAttachmentMetadata(raw.sys_id, client);

  return {
    // Identity
    ticket_id:      raw.number,
    external_sys_id: raw.sys_id,
    source:         'servicenow',
    tenant_id:      tenantId,

    // Content
    title:          raw.short_description?.value || '',
    description:    buildRichDescription(raw),   // combines description + work notes

    // Classification
    priority:       PRIORITY_MAP[raw.priority?.value] || 'P3',
    category:       CATEGORY_MAP[raw.category?.value] || 'general',
    urgency:        raw.urgency?.value,
    impact:         raw.impact?.value,

    // Services
    affected_services: affectedServices,
    business_service:  raw.business_service?.display_value,
    ci_name:           ciData?.name || raw.cmdb_ci?.display_value,
    environment:       ciData?.u_environment?.value || inferEnvironment(raw),

    // Metrics (if available from monitoring alert)
    metrics: {
      error_rate_pct: errorRate,
      latency_p99_ms: latencyP99,
      alert_source:   raw.u_alert_source?.value,
      correlation_id: raw.correlation_id?.value,
    },

    // People
    reporter_email:   raw.caller_id?.display_value,
    assignment_group: raw.assignment_group?.display_value,
    assigned_to:      raw.assigned_to?.display_value,

    // Attachments
    attachments:      attachments,

    // Timing
    opened_at:  raw.opened_at?.value,
    updated_at: raw.sys_updated_on?.value,

    // Raw
    raw_payload: raw,
  };
}

function buildRichDescription(raw: ServiceNowIncident): string {
  const parts: string[] = [];

  if (raw.description?.value) {
    parts.push('## Incident Description\n' + raw.description.value);
  }

  if (raw.work_notes?.value) {
    parts.push('## Work Notes\n' + raw.work_notes.value);
  }

  // Include correlation alert details if linked
  if (raw.correlation_id?.value) {
    parts.push(`## Alert Reference\nCorrelation ID: ${raw.correlation_id.value}`);
  }

  return parts.join('\n\n');
}

function parseAffectedServices(raw: ServiceNowIncident, ci: ServiceNowCI | null): string[] {
  const services = new Set<string>();

  // Custom field (most reliable if your team fills it in)
  if (raw.u_affected_services?.value) {
    raw.u_affected_services.value
      .split(',')
      .map(s => s.trim().toLowerCase())
      .filter(Boolean)
      .forEach(s => services.add(s));
  }

  // CI name (configuration item)
  if (ci?.name) services.add(ci.name.toLowerCase());

  // Business service
  if (raw.business_service?.display_value) {
    services.add(raw.business_service.display_value.toLowerCase());
  }

  // Regex extraction from description as last resort
  const descServices = extractServicesFromText(raw.description?.value || '');
  descServices.forEach(s => services.add(s));

  return Array.from(services);
}
```

---

## 5. Write-Back to ServiceNow

The agent writes back to ServiceNow at three points during a run.

### On run start
```typescript
async function onRunStart(run: AgentRun, ticket: NormalisedTicket): Promise<void> {
  await snClient.addWorkNote(
    ticket.external_sys_id,
    `[Agent System] Incident analysis started at ${new Date().toISOString()}\n` +
    `Run ID: ${run.run_id}\n` +
    `Agents: Triage + Memory scanning historical incidents\n` +
    `Track progress: ${process.env.UI_BASE_URL}/runs/${run.run_id}`
  );

  // Update assignment to agent system (optional — depends on your workflow)
  await snClient.updateIncident(ticket.external_sys_id, {
    assignment_group: process.env.SN_AGENT_GROUP_SYS_ID,  // "AI Agent System" group
    work_notes: 'Assigned to automated agent system for analysis'
  });
}
```

### On approval requested
```typescript
async function onApprovalRequired(run: AgentRun, approval: Approval): Promise<void> {
  await snClient.addWorkNote(
    run.sn_sys_id,
    `[Agent System] Human approval required before proceeding\n` +
    `Action: ${approval.action_description}\n` +
    `Risk: ${approval.risk_level}\n` +
    `Approve or reject: ${process.env.UI_BASE_URL}/approvals/${approval.approval_id}`
  );
}
```

### On resolution
```typescript
async function onRunResolved(run: AgentRun, episode: Episode): Promise<void> {
  await snClient.resolveIncident(run.sn_sys_id, {
    closeCode:  'Solved (Permanently)',
    notes:      buildResolutionNote(episode),
    rootCause:  episode.root_cause,
    action:     episode.resolution_action,
    runId:      run.run_id
  });
}

function buildResolutionNote(episode: Episode): string {
  return [
    `ROOT CAUSE: ${episode.root_cause}`,
    ``,
    `RESOLUTION: ${episode.resolution_text}`,
    ``,
    `STEPS TAKEN:`,
    episode.steps
      .filter(s => s.was_useful)
      .map(s => `  • ${s.tool_name}: ${s.outcome}`)
      .join('\n'),
    ``,
    `TIME TO RESOLVE: ${episode.resolution_mins} minutes`,
    `AGENT CONFIDENCE AT RESOLUTION: ${(episode.final_confidence * 100).toFixed(0)}%`,
    ``,
    `Full trace: ${process.env.UI_BASE_URL}/runs/${episode.run_id}`,
  ].join('\n');
}
```

---

## 6. Authentication Options

ServiceNow supports three auth methods. Use OAuth 2.0 in production.

```typescript
// Option A: Basic Auth (dev/testing only — never production)
const auth = {
  type: 'basic',
  username: process.env.SN_USERNAME,
  password: process.env.SN_PASSWORD,
};

// Option B: OAuth 2.0 Client Credentials (recommended for production)
const auth = {
  type: 'oauth2',
  tokenUrl: `${process.env.SN_BASE_URL}/oauth_token.do`,
  clientId: process.env.SN_CLIENT_ID,
  clientSecret: process.env.SN_CLIENT_SECRET,
  // Token is cached and refreshed automatically
};

// Option C: ServiceNow MID Server (for on-premise ServiceNow)
// Requires a MID server deployed in your network
// Contact ServiceNow for setup
```

---

## 7. Webhook Setup in ServiceNow

In ServiceNow admin console:

```
1. Navigate to: System Web Services → Outbound → REST Message
2. Create new REST Message: "TicketAgentWebhook"
   Endpoint: https://api.yourapp.com/webhooks/tickets/servicenow

3. Navigate to: System Policy → Business Rules
4. Create new Business Rule: "Send to TicketAgent on P1/P2 creation"
   Table: incident
   When: after insert
   Condition: priority IN ('1','2') AND state = '1'  (New P1 or P2)
   Script:
     var r = new sn_ws.RESTMessageV2('TicketAgentWebhook', 'post');
     r.setRequestBody(JSON.stringify({
       sys_id: current.sys_id.toString(),
       number: current.number.toString(),
       priority: current.priority.toString(),
       triggered_at: new GlideDateTime().toString()
     }));
     r.setRequestHeader('X-ServiceNow-Signature', gs.getProperty('ticket_agent.webhook_secret'));
     r.executeAsync();

5. Add another rule for escalation:
   Condition: priority CHANGED TO ('1','2')
```

---

## 8. Environment Variables Needed

```bash
# .env
SN_BASE_URL=https://yourcompany.service-now.com
SN_CLIENT_ID=your_oauth_client_id
SN_CLIENT_SECRET=your_oauth_client_secret
SN_AGENT_GROUP_SYS_ID=sys_id_of_ai_agent_group   # optional
SN_WEBHOOK_SECRET=random_32_char_string            # for webhook signature verification
```
