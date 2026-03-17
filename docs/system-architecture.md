# System Architecture

AI Company Platform v2 is a distributed system with three tiers: Control Plane (API), Execution Plane (Fly.io VMs), and Client (React SPA).

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       Web Browser (React)                       │
│  Dashboard, Team, Tasks, Approvals, Costs, Settings             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS + WebSocket
┌──────────────────────────▼──────────────────────────────────────┐
│                    Control Plane (NestJS API)                   │
│  ├─ Companies, Agents, Issues (CRUD)                            │
│  ├─ Heartbeat Scheduler (Task Scheduling)                       │
│  ├─ Execution Engine (Dispatch to VMs)                          │
│  ├─ Real-time Module (WebSocket, Redis pub/sub)                 │
│  └─ PostgreSQL Database (Multi-tenant)                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP + SSE + Redis pub/sub
        ┌──────────────────┼──────────────────┬──────────────────┐
        │                  │                  │                  │
    ┌───▼───┐          ┌───▼───┐         ┌───▼───┐          ┌───▼───┐
    │  VM-1 │          │  VM-2 │         │  VM-3 │          │  VM-N │
    │ Co-1  │          │ Co-2  │         │ Co-3  │          │ Co-N  │
    │┌─────┐│          │┌─────┐│         │┌─────┐│          │┌─────┐│
    ││ Exec││          ││ Exec││         ││ Exec││          ││ Exec││
    ││Engine││          ││Engine││         ││Engine││          ││Engine││
    │└─────┘│          │└─────┘│         │└─────┘│          │└─────┘│
    │ ┌───┐ │          │ ┌───┐ │         │ ┌───┐ │          │ ┌───┐ │
    │ │CLI│ │          │ │CLI│ │         │ │WS │ │          │ │GEN│ │
    │ │Ada│ │          │ │Ada│ │         │ │Ada│ │          │ │Ada│ │
    │ └───┘ │          │ └───┘ │         │ └───┘ │          │ └───┘ │
    └───────┘          └───────┘         └───────┘          └───────┘
    Fly.io VM          Fly.io VM          Fly.io VM         Fly.io VM
    (Auto-sleep)       (Auto-sleep)       (Auto-sleep)      (Auto-sleep)
```

**Principles:**
- **Multi-tenant:** Each company gets isolated VM + DB rows
- **Heartbeat model:** Discrete work windows, not continuous agents
- **Async dispatch:** API queues work; executor processes asynchronously
- **Real-time feedback:** WebSocket for live updates to dashboard

---

## Control Plane (API Server)

**Stack:** NestJS + Express + PostgreSQL (Neon) + Redis (Upstash)

### Components

#### 1. HTTP API (REST)
**Port:** 3000 (dev), exposed via Railway/Fly.io (prod)

**Routes:**
```
POST   /companies              Create company
GET    /companies/:id          Get company details
PUT    /companies/:id          Update company
DELETE /companies/:id          Delete company

POST   /companies/:id/agents   Create agent
GET    /companies/:id/agents   List agents
PUT    /agents/:id             Update agent config
DELETE /agents/:id             Terminate agent

POST   /companies/:id/issues   Create issue
GET    /companies/:id/issues   List issues
PUT    /issues/:id             Update issue status
POST   /issues/:id/comments    Add comment

POST   /approvals              Create approval request
GET    /approvals              List pending approvals
POST   /approvals/:id/approve  Approve request
POST   /approvals/:id/reject   Reject request

GET    /companies/:id/costs    Get cost breakdown
GET    /companies/:id/usage    Get usage metrics
```

**Authentication:**
- **User sessions:** Better Auth (email/password)
- **API keys:** Company-scoped for external integrations
- **Agent JWT:** Issued by API, used by executor for callback auth

#### 2. Heartbeat Scheduler
**Framework:** @nestjs/schedule
**Trigger:** Cron-like intervals per agent

**Flow:**
```
Scheduler fires every 5 min
  ↓
Query: which agents have a heartbeat due?
  ↓
For each agent:
  - Check budget (skip if exceeded)
  - Check for issues assigned
  - Queue execution request
  ↓
Execution Engine processes queue
```

**Execution Window:**
```typescript
// Pseudocode
async executeHeartbeat(agentId: string) {
  const agent = await getAgent(agentId);
  if (agent.spentAmount >= agent.budgetLimit) {
    agent.status = 'PAUSED'; // Budget exceeded
    return;
  }

  const issues = await getUnassignedIssues(agent.companyId);
  if (issues.length === 0) {
    return; // Nothing to do
  }

  const issue = issues[0]; // Next task
  await checkoutIssue(issue.id, agent.id); // Atomic

  const dispatch = new ExecutionRequest({
    agentId: agent.id,
    issueId: issue.id,
    prompt: buildPrompt(agent, issue),
    vmUrl: getVmUrl(agent.companyId),
  });

  await executionEngine.dispatch(dispatch);
}
```

**Frequency:** Every 5 minutes by default (configurable per agent)

#### 3. Execution Engine Service
**Purpose:** Route work requests to correct Fly.io VM, stream results back

**Flow:**
```
1. Heartbeat scheduler queues ExecutionRequest
2. Execution Engine:
   a. Gets company VM URL (Fly.io)
   b. Routes to correct adapter (Claude, OpenClaw, Process)
   c. POSTs request to executor/execute
   d. Streams SSE response back to client (via WebSocket)
   e. Calculates costs, updates database
3. Returns ExecutionResult
```

**Executor Routing:**
```typescript
// Which adapter to use?
const getAdapterType = (agent: Agent): AdapterType => {
  switch (agent.adapterType) {
    case 'CLAUDE':
      return new ClaudeCliAdapter();
    case 'OPENCLAW':
      return new OpenClawWebSocketAdapter();
    case 'PROCESS':
      return new ProcessGenericAdapter();
    default:
      throw new UnknownAdapterException();
  }
};
```

#### 4. Real-time Module (WebSocket + Pub/Sub)
**Technology:** Socket.io or raw WebSocket + Redis pub/sub

**Channels:**
```
/companies/{id}                   → Company status, agent updates
/companies/{id}/agents/{id}       → Agent execution status
/companies/{id}/issues/{id}       → Task progress, comments
/companies/{id}/approvals         → New approval requests
/companies/{id}/costs             → Real-time spending
```

**Event Examples:**
```json
{
  "type": "agent_status_changed",
  "agentId": "agent-123",
  "status": "executing",
  "timestamp": "2026-03-17T12:00:00Z"
}

{
  "type": "issue_updated",
  "issueId": "issue-456",
  "status": "in_progress",
  "assignedTo": "agent-123",
  "progress": "40%"
}

{
  "type": "execution_event",
  "runId": "run-789",
  "eventType": "tool_call",
  "toolName": "write_file",
  "data": { "path": "/workspace/index.ts", "lines": 250 }
}

{
  "type": "cost_update",
  "companyId": "company-1",
  "costDelta": 0.12,
  "totalSpent": 1250.50
}
```

#### 5. Database (PostgreSQL)
**Host:** Neon managed PostgreSQL
**Replication:** Automatic (Neon)
**Backup:** Automated daily

**Schema:** 35+ tables (see codebase-summary.md)

**Key Characteristics:**
- **Multi-tenant:** Every table filtered by `companyId`
- **JSONB fields:** Flexible configs (adapterConfig, runtimeConfig, usageJson)
- **Event logging:** heartbeat_run_events for full execution trace
- **Indexes:** On companyId, agentId, createdAt for fast queries

#### 6. Redis (Upstash)
**Purpose:** Pub/sub for real-time events, caching

**Use Cases:**
- Publish heartbeat events (API → WebSocket clients)
- Cache company config (avoid repeated DB queries)
- Cache agent list (invalidated on agent changes)
- Session store (if needed for Better Auth)

---

## Execution Plane (Fly.io VMs)

**Stack:** Express.js + Node.js

### Per-Company VM
One VM per company. Specifications:
- **Instance:** Fly.io Machine (shared-cpu-1x, 256MB RAM)
- **Uptime:** Auto-hibernates after 5 min idle, wakes on request
- **Storage:** 10GB persistent disk (workspace)
- **Restart policy:** Always-restart on crash

### Executor Service (Express)
**Port:** 8080

**Endpoints:**
```
POST /execute                          Trigger execution
GET  /status                           Agent status
GET  /logs?runId=:id&lines=:count      Execution logs
GET  /workspace/:path                  Read workspace file
```

**Request Flow:**
```
POST /execute
{
  agentId: "agent-123",
  issueId: "issue-456",
  runId: "run-789",
  adapterType: "CLAUDE_CLI",
  prompt: "Write a function to...",
  context: {
    companyGoal: "...",
    agentRole: "Engineer",
    agentPermissions: ["code_write", "file_read"],
    recentHistory: [...]
  }
}
  ↓
1. Validate request (JWT auth, companyId match)
2. Create RunRecord (track execution)
3. Get AdapterRegistry.get(adapterType)
4. Adapter.execute(request)
   - Spawn process or open WebSocket
   - Send prompt
   - Stream stdout/stderr
   - Capture tool calls
5. Parse output
6. Calculate cost (tokens, compute time)
7. Stream SSE back to API
8. Return ExecutionResult
```

### Adapter System

**Interface:**
```typescript
interface IAdapter {
  execute(request: ExecutionRequest): Promise<ExecutionResult>;
  estimateCost(request: ExecutionRequest): Promise<CostEstimate>;
}

interface ExecutionRequest {
  agentId: string;
  issueId: string;
  runId: string;
  prompt: string;
  context: ExecutionContext;
  timeout: number; // ms
}

interface ExecutionResult {
  success: boolean;
  output: string;
  toolCalls: ToolCall[];
  costUSD: number;
  tokensUsed: number;
  executionMs: number;
  error?: string;
}
```

#### V1 Adapters

**1. Claude CLI Adapter**
```
Subprocess spawn:
  $ claude-cli --task "your prompt" [--input file.txt]
  ↓
Capture stdout (agent output)
Capture stderr (errors)
Parse tool calls from output
Count tokens (via Claude API response)
```

**2. OpenClaw WebSocket Adapter**
```
WebSocket connect: wss://api.openclaw.dev
  ↓
Send JSON message:
  {
    "type": "execute_task",
    "prompt": "...",
    "context": {...}
  }
  ↓
Stream response (bidirectional)
Handle tool calls in real-time
Graceful disconnect
```

**3. Process Generic Adapter**
```
Subprocess spawn:
  $ /path/to/executable
  ↓
Pass stdin: JSON request
  {
    "type": "execute",
    "prompt": "...",
    "context": {...}
  }
  ↓
Capture stdout: JSON response
Parse tool calls
Timeout after 60s
```

### Workspace Management
**Location:** `/home/workspace/` on VM

```
/home/workspace/
├── git/                    Git repositories
│   ├── repo-1/
│   ├── repo-2/
│   └── ...
├── files/                  Other files (documents, configs)
├── memory/                 Agent memory (JSON)
│   ├── conversation.jsonl  Chat history
│   ├── issues-done.json    Completed tasks
│   └── metrics.json        Performance data
└── logs/                   Execution logs
    └── run-{id}.log        One log per heartbeat
```

**Persistence:**
- Survives VM hibernation (disk is persistent)
- Survives process restarts
- Cleared on company deletion

---

## Communication Patterns

### API → Executor (Sync + Async)

**Synchronous (with timeout):**
```
API: POST http://vm-123.internal:8080/execute
Body: ExecutionRequest
  ↓
Executor waits up to 30s
  ↓
Returns ExecutionResult (or timeout error)
```

**Asynchronous (via SSE):**
```
API: POST /execute (same request)
  ↓
Executor queues work
Returns: SSE stream endpoint
  ↓
API opens SSE: GET /execute/{runId}/stream
  ↓
Executor streams events:
  data: {"type": "started", "timestamp": "..."}
  data: {"type": "tool_call", "toolName": "write_file", ...}
  data: {"type": "completed", "output": "...", "costUSD": 0.12}
  ↓
Stream closes
```

### API → Client (WebSocket)

**Real-time updates:**
```
Client: WebSocket /ws?token={jwt}&companyId={id}
  ↓
Subscribe to channel: /companies/{id}/execution
  ↓
API publishes to Redis:
  PUBLISH company-1:execution {"type": "agent_status", "status": "executing"}
  ↓
Redis → API → WebSocket → Client (all subscribed clients)
```

### Agent Callbacks (Executor → API)

**Agent can call API during execution:**
```
Agent (in prompt): "POST /api/issues/{id}/comment with your findings"
  ↓
Executor provides HTTP client in context
  ↓
Agent makes request with JWT (issued by API)
  ↓
API validates JWT (contains agentId, companyId)
  ↓
Processes request (authorization check)
  ↓
Returns result to agent
```

**JWT Format:**
```json
{
  "type": "agent",
  "agentId": "agent-123",
  "companyId": "company-1",
  "permissions": ["issue_read", "issue_write", "comment_write"],
  "exp": 1710680000
}
```

---

## Data Flow

### Create Issue & Assign to Agent

```
1. User creates issue (UI form)
   POST /companies/:id/issues
   → Stored in DB (status: backlog)

2. User assigns to agent
   PUT /issues/:id { assignedTo: "agent-123" }
   → Issue status: todo
   → Publish WebSocket event

3. Next heartbeat (~5 min)
   Scheduler: "Check agent-123 for work"
   → Query: issues WHERE assignedTo = agent-123 AND status = todo
   → Found 1 issue
   → Execute heartbeat

4. Execution Engine:
   a. POST http://vm-company-1:8080/execute
   b. Wait for result (timeout 30s)
   c. Receive ExecutionResult
   d. Update issue status → in_progress
   e. Store heartbeat_run (success/failure)
   f. Update costs (add costUSD to company)
   g. Publish: type=issue_updated, status=in_progress

5. Client receives WebSocket:
   {type: "issue_updated", status: "in_progress"}
   → Dashboard refreshes task board in real-time

6. Agent completes work
   ExecutionResult { success: true, output: "..." }
   → Issue status → done
   → Publish final event

7. Client shows: ✅ Task completed, cost $0.12
```

### Multi-tenant Cost Tracking

```
ExecutionResult from executor:
{
  success: true,
  tokensUsed: 2500,
  executionMs: 3200,
  costUSD: 0.12
}
  ↓
API receives result:
1. Create cost_events row
   {
     companyId: "company-1",
     agentId: "agent-123",
     runId: "run-789",
     tokenCount: 2500,
     computeCostUSD: 0.02,
     llmCostUSD: 0.10,
     totalCostUSD: 0.12
   }
2. Update agent total: agent.spentAmount += 0.12
3. Check budget: if (spentAmount > budgetLimit) agent.status = PAUSED
4. Publish WebSocket: {type: cost_update, costDelta: 0.12, totalSpent: 1250.50}
5. Store in monthly_billing for invoice
```

---

## Security Model

### Authentication & Authorization

**User Auth (Better Auth):**
- Email + password or OAuth
- Session cookie (HttpOnly, Secure)
- Token refresh on expiry

**Agent Auth (JWT):**
- Issued by API when executor requests execution
- Short-lived (10 min expiry)
- Contains: agentId, companyId, permissions
- Executor uses JWT for API callbacks

**API Key Auth (Company):**
- Generated by user (long random string)
- Hashed in DB
- Scoped to company
- Used for external integrations

### Multi-tenant Isolation

**Every query filters by companyId:**
```typescript
// ✅ Safe
const agents = await db.query.agents.findMany({
  where: eq(agents.companyId, userCompanyId), // Always include
});

// ❌ Dangerous (data leak)
const agents = await db.query.agents.findMany();
```

**VM Isolation:**
- Each company has isolated Fly.io Machine
- No shared storage between companies
- No cross-company API keys

**Database Row-level Security:**
- RLS policies on sensitive tables
- Column encryption for secrets (API keys)

### API Key Vault

**Storage:**
```
company_api_keys table:
{
  id: uuid,
  companyId: uuid,
  platform: "openai" | "anthropic" | "...",
  encryptedKey: string,  // AES-256-GCM encrypted
  keyHash: string,       // Indexed for lookup
  createdAt: timestamp,
  lastUsedAt: timestamp,
  expiresAt: timestamp
}
```

**Encryption:**
- Key: Stored in environment variable (ENCRYPTION_KEY)
- Algorithm: AES-256-GCM
- Decryption on-the-fly (never logged)

**Rotation:**
- Manual: User generates new key
- Automatic: Revoke after 90 days (warning at 80 days)

---

## Scaling & Performance

### Horizontal Scaling

**API Server (Stateless):**
- Deploy multiple instances (Railway/Fly.io)
- Load balance via Fly.io Router or external LB
- All state in PostgreSQL/Redis (shareable)
- Horizontally scale to handle traffic

**Executor VMs (Auto-scaling):**
- One per company (multi-tenant isolation)
- Auto-hibernates when idle (saves cost)
- Wakes on first heartbeat (cold start ~2-3s)
- Multiple agents per VM (shared executor)

**Database (Managed):**
- Neon handles replication, failover
- Connection pooling: PgBouncer
- Query optimization: Indexes on companyId, agentId, createdAt

### Caching Strategy

**Redis Cache:**
- **Company config:** Cache 10 min, invalidate on update
- **Agent list:** Cache 10 min, invalidate on agent changes
- **Cost totals:** Cache 5 min, recalculate on cost event
- **Pub/sub buffer:** Keep 100 recent events per company

**API Response Caching:**
- **GET /companies/:id:** Cache 5 min (private)
- **GET /companies/:id/agents:** Cache 3 min
- **GET /companies/:id/costs:** Cache 1 min (real-time)

### Database Optimization

**Indexing:**
```sql
CREATE INDEX idx_agents_company_id ON agents(company_id);
CREATE INDEX idx_issues_company_id ON issues(company_id);
CREATE INDEX idx_issues_assigned_to ON issues(assigned_to);
CREATE INDEX idx_heartbeat_runs_agent_id ON heartbeat_runs(agent_id, created_at DESC);
CREATE INDEX idx_cost_events_company_id ON cost_events(company_id, created_at DESC);
```

**Query Optimization:**
- Use EXPLAIN ANALYZE for slow queries
- Batch inserts (cost_events) every 10s
- Denormalize frequently accessed fields (agent.spentAmount)

---

## Fault Tolerance & Recovery

### Executor Crashes
```
If executor crashes mid-execution:
1. Heartbeat fails (timeout after 30s)
2. API retries with exponential backoff (1s, 2s, 4s, max 30s)
3. After 3 retries, mark run as FAILED
4. Issue remains in previous status (e.g., todo)
5. User notified via dashboard + email
6. Manual retry button available
```

### VM Crashes
```
If company VM crashes:
1. Fly.io auto-restarts (always-restart policy)
2. Executor service re-initializes
3. Workspace persists (on disk)
4. Pending executions queued in API
5. When VM is back up, executions resume
```

### Database Connection Loss
```
If DB connection drops:
1. API returns 503 Service Unavailable
2. NestJS handles reconnection (automatic pooling)
3. In-flight requests buffered in Redis
4. Clients receive error and can retry
5. No data loss (transactions rolled back)
```

### Cost Tracking Failures
```
If cost calculation fails:
1. ExecutionResult still processed (success tracked)
2. Cost event creation retried (queue in Redis)
3. Cost catch-up job runs every minute
4. Missing costs reconciled from logs
```

---

## Deployment Architecture

```
┌──────────────────────────────────────────┐
│     Neon PostgreSQL (Managed)            │
│     - Auto-backup daily                  │
│     - Connection pooling (PgBouncer)     │
│     - Read replicas available            │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│     Upstash Redis (Managed)              │
│     - Pub/sub for real-time events       │
│     - Cache for config/agents            │
│     - Session storage                    │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│     Control Plane (Railway or Fly.io)    │
│     ├─ NestJS API (node:20 Alpine)       │
│     ├─ Scheduler (@nestjs/schedule)      │
│     └─ WebSocket (Socket.io or raw)      │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│     Frontend (Vercel or Fly.io)          │
│     ├─ React 19 SPA (Vite build)         │
│     └─ Tailwind CSS + shadcn/ui          │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│     Fly.io Machines (Per-company VMs)    │
│     ├─ Executor service (Node.js)        │
│     ├─ Workspace (persistent disk)       │
│     └─ Adapters (Claude, OpenClaw, etc)  │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│     S3 (File Storage)                    │
│     ├─ Workspace backups                 │
│     ├─ Execution logs (archived)         │
│     └─ Company exports                   │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│     Monitoring & Logging                 │
│     ├─ Axiom (logs)                      │
│     ├─ Sentry (errors)                   │
│     ├─ Datadog (metrics)                 │
│     └─ GitHub Actions (CI/CD)            │
└──────────────────────────────────────────┘
```

---

## Example: Full Task Execution Sequence

```
T=0s
┌─ User creates issue in dashboard
│  POST /companies/co-1/issues
│  { title: "Write homepage", assigned_to: "agent-eng-1" }
└─ Status: todo, assigned_to: agent-eng-1

T=5m (Next heartbeat)
┌─ Scheduler fires for agent-eng-1
│  Query: SELECT * FROM issues WHERE company_id = co-1 AND assigned_to = agent-eng-1 AND status = todo
│  Found: 1 issue (id: issue-456)
│  Check budget: agent.spent (50) < agent.limit (500) ✓
│  Checkout: UPDATE issues SET status = in_progress, checked_out_by = agent-eng-1

T=5m + 100ms
┌─ Execution Engine.dispatch()
│  VM URL: fly-vm-co-1.internal:8080
│  POST http://fly-vm-co-1.internal:8080/execute
│  {
│    agentId: "agent-eng-1",
│    issueId: "issue-456",
│    runId: "run-789",
│    adapterType: "CLAUDE_CLI",
│    prompt: "You are an engineer. Complete this task: Write homepage\n\nContext: ...",
│    context: { companyGoal: "...", agentRole: "engineer", ... }
│  }

T=5m + 150ms
┌─ Executor receives request
│  JWT validation ✓ (agentId + companyId match)
│  Get adapter: ClaudeCliAdapter
│  Spawn: $ claude-cli --task "..."
│  Stream stdout/stderr
│  Parse tool calls
│  Track execution time: 8500ms
│  Count tokens: 3200 input + 1500 output = 4700 total
│  Estimate cost: (4700 / 1M) * $3 = $0.0141
│  Store RunRecord (status: completed)

T=5m + 8.7s
┌─ Executor returns ExecutionResult
│  {
│    success: true,
│    output: "Created index.tsx with responsive design...",
│    toolCalls: [{name: "write_file", path: "/workspace/index.tsx", ...}],
│    tokensUsed: 4700,
│    executionMs: 8500,
│    costUSD: 0.0141
│  }

T=5m + 8.8s
┌─ API receives result
│  1. Create cost_event row (company_id: co-1, cost: 0.0141)
│  2. Update agent: spent += 0.0141
│  3. Update issue: status = done, completed_at = now
│  4. Update heartbeat_run: status = completed
│  5. Publish Redis: PUBLISH co-1:execution { type: "issue_updated", status: "done" }

T=5m + 8.9s
┌─ Client receives WebSocket event
│  {type: "issue_updated", issueId: "issue-456", status: "done", costUSD: 0.0141}
│  Dashboard updates: Task shows as ✅ Done, cost $0.01

T=5m + 9s
└─ User sees live update in dashboard
   - Issue moved to "Done" column
   - Cost updated: $0.01 total
   - Comments/output visible in detail view
```

---

**Version:** 1.0
**Last Updated:** 2026-03-17
**Diagrams:** Mermaid + ASCII
**See Also:** `./blueprint/03-architecture/`, `./project-overview-pdr.md`
