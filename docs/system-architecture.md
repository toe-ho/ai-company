# System Architecture

## High-Level Overview

```
                         User (Browser)
                              │
                    ┌─────────┴──────────┐
                    │     React UI        │
                    │  (Vite, Tailwind)   │
                    └─────────┬──────────┘
                              │ HTTPS / WebSocket
                              │ (session cookie)
                              │
                    ┌─────────┴──────────────────────┐
                    │   CONTROL PLANE                 │
                    │   (NestJS API + Scheduler)      │
                    ├────────────────────────────────┤
                    │  ┌─────────────────────────┐   │
                    │  │ Auth Layer (Phase 2)    │   │
                    │  │ ├─ Better Auth sessions │   │
                    │  │ ├─ BoardAuthGuard       │   │
                    │  │ ├─ AgentAuthGuard       │   │
                    │  │ └─ CompanyAccessGuard   │   │
                    │  ├─ HTTP Controllers       │   │
                    │  │ ├─ board/ (users)       │   │
                    │  │ ├─ agent/ (callbacks)   │   │
                    │  │ ├─ public/ (auth)       │   │
                    │  │ └─ internal/ (health)   │   │
                    │  ├─ CQRS Bus              │   │
                    │  │ ├─ Commands             │   │
                    │  │ └─ Queries              │   │
                    │  ├─ Heartbeat Scheduler   │   │
                    │  │ └─ @nestjs/schedule    │   │
                    │  ├─ Execution Engine      │   │
                    │  ├─ Services              │   │
                    │  │ ├─ ExecutionEngine      │   │
                    │  │ ├─ Provisioner          │   │
                    │  │ ├─ ApiKeyVault          │   │
                    │  │ ├─ LiveEvents          │   │
                    │  │ └─ Storage (S3)         │   │
                    │  └─ Real-time Events      │   │
                    │     └─ Redis pub/sub      │   │
                    └──┬─────────────┬──────────┘
                       │             │
            ┌──────────┘             └──────────┐
            │                                    │
  ┌─────────┴──────────┐          ┌─────────────┴────────────┐
  │  PostgreSQL (Neon)  │          │  EXECUTION PLANE          │
  │  Multi-tenant DB    │          │  (Fly.io Machines)        │
  │  35+ tables         │          │                           │
  └─────────────────────┘          │  ┌─────────────────────┐  │
                                   │  │ Company A VM         │  │
            ┌──────────┐           │  │ ├── Agent Executor   │  │
            │ Redis     │           │  │ ├── Agent processes  │  │
            │ (Upstash) │           │  │ ├── Git workspace    │  │
            │ pub/sub   │           │  │ └── Persistent vol   │  │
            └──────────┘           │  └─────────────────────┘  │
                                   │                           │
            ┌──────────┐           │  ┌─────────────────────┐  │
            │ S3        │           │  │ Company B VM         │  │
            │ Storage   │           │  │ ├── Agent Executor   │  │
            │           │           │  │ └── ...              │  │
            └──────────┘           │  └─────────────────────┘  │
                                   └──────────────────────────┘

       Agent processes call BACK to Control Plane API
       using injected JWT + API key + environment variables
```

## Control Plane vs Execution Plane

### Control Plane
**Location:** Cloud-hosted (Railway or Fly.io)
**Purpose:** Orchestrate everything. Schedule execution. Track costs. Manage auth & governance.
**Components:**
- NestJS HTTP API (REST endpoints)
- @nestjs/schedule (heartbeat scheduler)
- Drizzle ORM (database access)
- CQRS bus (command/query dispatch)
- Service layer (Provisioner, ExecutionEngine, ApiKeyVault, LiveEvents)
- WebSocket gateway (real-time updates)

**Scalability:** Stateless. Horizontal scaling (multiple API server replicas behind load balancer).

### Execution Plane
**Location:** Per-company Fly.io Machine
**Purpose:** Run agent processes in isolated environment.
**Components:**
- Agent Executor (HTTP server accepting execution requests)
- Agent processes (Claude, GPT, Gemini, etc. running in subprocess)
- Workspace (git repos, files, persistent volumes)
- Adapter implementations (convert request to provider API)

**Scalability:** Per-company. 1 VM per company (agents within company share VM). Auto-hibernates when idle.

**Why Split?**
1. **Isolation:** Company A's agents can't see Company B's workspace
2. **Cost efficiency:** Agents run only when needed (heartbeat windows)
3. **Independent scaling:** Control plane scales horizontally; VMs scale per-company
4. **Failure containment:** Agent crash doesn't affect other companies

## Component Communication

### User → Control Plane
**Protocol:** HTTPS + WebSocket
**Auth:** Session cookies (Better Auth)
**Examples:**
- `POST /api/companies` — Create company
- `GET /api/agents` — List agents
- `POST /api/issues/{id}/checkout` — Checkout task
- WebSocket `/api/realtime` — Subscribe to live events

### Control Plane → Database
**Protocol:** TCP (PgBouncer connection pool)
**ORM:** Drizzle ORM (type-safe)
**Multi-tenant:** Every query filters by `companyId`
**Example:**
```typescript
const agents = await db.query.agents.findMany({
  where: eq(agents.companyId, companyId),
});
```

### Control Plane → Execution Plane
**Protocol:** HTTP POST + SSE (Server-Sent Events)
**Auth:** None (both within VPC, or IP-restricted)
**Flow:**
1. Control plane builds `ExecutionRequest` (context, API keys, agent config)
2. POSTs to Agent Executor: `POST http://company-vm:3200/execute`
3. Agent Executor spawns agent process, starts streaming output
4. Control plane reads SSE stream: `{ type: "stdout", data: "..." }`
5. For each event: persist `HeartbeatRunEvent`, publish live event

### Execution Plane → Control Plane (Agent Callbacks)
**Protocol:** HTTPS
**Auth:** JWT + API key (injected at execution time)
**Examples:**
- `POST /api/agent/issues/{id}/checkout` — Agent claims task
- `POST /api/agent/issues/{id}/comment` — Agent posts update
- `POST /api/agent/issues/{id}/update-status` — Agent marks done
- `X-Run-Id` header links action back to specific heartbeat run

### Real-time Updates
**Protocol:** Redis pub/sub + WebSocket
**Flow:**
1. Something happens (agent paused, task assigned, cost event)
2. Control plane publishes event to Redis channel: `companies/{companyId}/events`
3. WebSocket gateway subscribes to Redis
4. On message, broadcasts to all connected browser clients
5. Dashboard updates in real-time (<1s latency)

## Heartbeat Execution Flow

```
1. SCHEDULER TICK (every 30 seconds)
   ├─ Acquire pg advisory lock (only 1 replica runs)
   ├─ Reap orphaned runs (>5 min without update)
   ├─ Resume queued runs
   ├─ Tick timers: for each agent, check if heartbeat interval elapsed
   └─ If yes, enqueue agent for heartbeat

2. CONTROLLER: POST /api/companies/{companyId}/agents/{agentId}/heartbeat
   └─ Calls: commandBus.execute(new InvokeHeartbeatCommand(...))

3. INVOKE HEARTBEAT HANDLER
   ├─ Retrieve agent, company, budget check
   ├─ If over budget: throw BudgetExceededException, stop
   ├─ Get LLM API key from vault (decrypt)
   ├─ Ensure VM is running (wake if hibernated)
   └─ Call: executionEngine.execute(request)

4. EXECUTION ENGINE SERVICE
   ├─ Build ExecutionRequest (context, session, API key, agent JWT)
   ├─ POST to Agent Executor: http://company-vm:3200/execute
   ├─ Start reading SSE stream (stdout, stderr, done)
   └─ Yield events back to caller

5. INVOKE HANDLER (continued)
   └─ For each event from stream:
      ├─ Persist HeartbeatRunEvent to DB
      ├─ Publish live event to Redis (companyId/events channel)
      └─ If agent made API call: parse output, record cost

6. ON COMPLETION
   ├─ Record CostEvent (total tokens, compute time)
   ├─ Update AgentRuntimeState (lastHeartbeatAt, session codec)
   ├─ Check budget again: if over, pause agent
   ├─ Schedule VM hibernation (10 min idle timeout)
   └─ Return results to caller
```

## Agent Execution on Fly.io VM

```
Agent Executor (HTTP Server)
        ↓
POST /execute {
  "agentId": "...",
  "context": { goals, tasks, workspace, history },
  "sessionCodec": { serialized state },
  "apiKey": "eyJhbGc..." (JWT),
  "env": { API_URL, ANTHROPIC_API_KEY, ... }
}
        ↓
├─ Deserialize agent session from sessionCodec
├─ Set environment variables
├─ Spawn subprocess: node adapt-claude.js
│  ├─ Read SKILL.md (agent instructions)
│  ├─ Call LLM API (Anthropic, OpenAI, etc.)
│  ├─ Agent thinks & decides
│  ├─ Agent calls tools (git push, task checkout, write file, etc.)
│  └─ Stream stdout/stderr back
└─ On exit:
   ├─ Serialize agent session (save state)
   ├─ Return serialized session back to control plane
   └─ Control plane saves to agentRuntimeState
```

## Key Services

### ExecutionEngineService
Dispatches execution requests to Fly.io VM and streams results back.

```typescript
interface IExecutionEngineService {
  execute(request: ExecutionRequest): AsyncIterable<ExecutionEvent>;
  cancel(runId: string): Promise<void>;
}

// Implementation
async *execute(request: ExecutionRequest) {
  const vm = await this.provisioner.ensureVm(request.companyId);
  const response = await fetch(`http://${vm.ip}:3200/execute`, {
    method: "POST",
    body: JSON.stringify(request),
  });

  const reader = response.body!.getReader();
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const event = parseSSEEvent(value);
    yield event;
  }
}
```

### ProvisionerService
Manages per-company Fly.io VM lifecycle (boot, wake, hibernate, destroy).

```typescript
interface IProvisionerService {
  ensureVm(companyId: string): Promise<VmInfo>;
  hibernateVm(companyId: string): Promise<void>;
  destroyVm(companyId: string): Promise<void>;
}

// VM state transitions
stopped → starting → running ↔ hibernating → stopped
```

### ApiKeyVaultService
Encrypts/decrypts LLM API keys. Keys never leave control plane unencrypted.

```typescript
interface IApiKeyVaultService {
  store(companyId: string, provider: string, key: string): Promise<void>;
  retrieve(companyId: string, provider: string): Promise<string>;
  validate(companyId: string, provider: string): Promise<boolean>;
  revoke(companyId: string, provider: string): Promise<void>;
}

// AES-256 encryption at rest
Key: master_key (from env)
Cipher: AES-256-GCM
IV: random per key
Store: encrypted_key + iv (in DB)
```

### LiveEventsService
Publishes events to Redis for real-time dashboard updates.

```typescript
interface ILiveEventsService {
  publish(companyId: string, event: LiveEvent): Promise<void>;
  subscribe(
    companyId: string,
    callback: (event: LiveEvent) => void,
  ): () => void;
}

// Redis channels: companies/{companyId}/events
// Event types: agent-paused, task-assigned, cost-event, etc.
```

### StorageService
S3 operations for logs, attachments, backups.

```typescript
interface IStorageService {
  putObject(key: string, body: Buffer): Promise<void>;
  getObject(key: string): Promise<Buffer>;
  deleteObject(key: string): Promise<void>;
}

// Paths: companies/{companyId}/logs/{runId}.json
//        companies/{companyId}/attachments/{...}
```

## Module Structure (NestJS)

```
src/
├── shared/               ← @Global() module
│   ├── repositories/     ← All repository implementations
│   ├── services/         ← All cross-cutting services
│   ├── commands/         ← All command handlers
│   ├── queries/          ← All query handlers
│   ├── events/           ← Domain event handlers
│   └── shared.module.ts  ← @Global()
│
├── api/                  ← HTTP entry points
│   ├── controllers/
│   │   ├── board/        ← /api/board/* (user endpoints)
│   │   ├── agent/        ← /api/agent/* (agent callbacks)
│   │   ├── public/       ← /api/public/* (auth, signup)
│   │   └── internal/     ← /api/internal/* (health, etc.)
│   ├── dtos/             ← Request/response DTOs
│   ├── guards/           ← Auth guards
│   ├── decorators/       ← @CurrentActor, @CompanyId, etc.
│   ├── interceptors/     ← Logging, scope injection
│   ├── filters/          ← Exception filter
│   ├── pipes/            ← ZodValidationPipe
│   └── api.module.ts
│
├── scheduler/            ← Heartbeat scheduler
│   ├── heartbeat.service.ts
│   ├── scheduler.module.ts
│   └── scheduled.events.ts
│
├── realtime/             ← WebSocket gateway
│   ├── gateway.ts        ← WebSocket @WebSocketGateway
│   ├── realtime.service.ts
│   └── realtime.module.ts
│
├── domain/               ← Domain layer (no framework)
│   ├── entities/         ← Company, Agent, Issue, etc.
│   ├── repositories/     ← ICompanyRepository, IAgentRepository, etc.
│   ├── enums/            ← AgentStatus, IssueStatus, etc.
│   └── exceptions/       ← BudgetExceededException, etc.
│
├── infrastructure/       ← Database & external services
│   ├── database/
│   │   ├── schemas/      ← Drizzle schemas (companies, agents, etc.)
│   │   └── db.ts         ← Drizzle client
│   ├── mappers/          ← Row → Entity mappers
│   ├── clients/          ← Fly.io client, S3 client, etc.
│   └── infrastructure.module.ts
│
└── main.ts               ← App bootstrap
```

## Database Schema (Simplified)

```
users (Better Auth)
├─ id, name, email, emailVerified, plan

companies
├─ id (UUID)
├─ ownerId (FK → users)
├─ name, description, status
├─ budgetMonthlyCents, spentMonthlyCents
├─ companyId (FK → companies.id, for multi-level orgs)

agents
├─ id (UUID)
├─ companyId (FK → companies)
├─ name, role, status
├─ reportsTo (FK → agents.id, org tree)
├─ adapterType, adapterConfig (JSONB)
├─ budgetMonthlyCents, spentMonthlyCents
├─ lastHeartbeatAt

issues (tasks)
├─ id (UUID)
├─ companyId (FK → companies)
├─ number (auto-increment per company)
├─ title, description, status, priority
├─ assignedTo (FK → agents)
├─ checkedOutBy (FK → agents)
├─ parentId (FK → issues.id, subtasks)

heartbeatRuns
├─ id (UUID)
├─ agentId (FK → agents)
├─ status (pending, running, done, failed)
├─ startedAt, completedAt

heartbeatRunEvents
├─ id (UUID)
├─ heartbeatRunId (FK → heartbeatRuns)
├─ type (stdout, stderr, tool-call, done)
├─ data (JSONB)

costEvents
├─ id (UUID)
├─ agentId (FK → agents)
├─ heartbeatRunId (FK → heartbeatRuns)
├─ provider (anthropic, openai, etc.)
├─ tokensUsed, costCents

companyApiKeys (encrypted)
├─ id (UUID)
├─ companyId (FK → companies)
├─ provider (anthropic, openai, etc.)
├─ encryptedKey (AES-256)

agentRuntimeState
├─ agentId (FK → agents)
├─ sessionCodec (JSONB, serialized state)
├─ updatedAt
```

## Security Model

### Authentication
- **Users:** Better Auth (session cookies)
- **Agents:** JWT (issuer: control plane, audience: company)
- **Cross-origin:** CORS whitelist

### Authorization
- **BoardAuthGuard:** Validates session cookie
- **AgentAuthGuard:** Validates JWT or API key
- **CompanyAccessGuard:** Verifies actor owns the company
- **CompanyRoleGuard:** Enforces role-based access (owner, admin, viewer)

### Encryption
- **API Keys:** AES-256-GCM (master key from env)
- **Transport:** HTTPS only
- **At Rest:** All secrets encrypted in database

### Multi-tenant Isolation
- **Query Filtering:** Every query includes `WHERE companyId = ?`
- **Row-level Security:** DB enforces company isolation
- **No data leakage:** Audit logs reviewed

### Cost Control
- **Per-agent budget:** Hard cap, no execution beyond
- **Per-company budget:** Aggregate of agents
- **Monthly reset:** Automatic on 1st of month
- **Alerts:** 80%, 100% budget warnings

### Rate Limiting
- **Public endpoints:** 100 req/min per IP
- **Authenticated endpoints:** 1000 req/min per user
- **Agent endpoints:** 500 req/min per API key

## Monitoring & Observability

### Logging
- **Format:** Structured JSON (Pino)
- **Levels:** debug, info, warn, error
- **Destinations:** stdout (captured by cloud provider)

### Metrics
- **Request count/latency:** Via HTTP interceptor
- **Database queries:** Via Drizzle hook
- **Cost events:** All recorded in costEvents table
- **Agent health:** lastHeartbeatAt, errorCount

### Tracing
- **Run ID:** `X-Run-Id` header links all actions to heartbeat
- **Activity log:** All mutations recorded with actor, timestamp, details
- **Audit trail:** Who did what, when, on which company

### Alerting
- **Budget exceeded:** Notify user, pause agent
- **VM creation failed:** Retry with backoff, alert ops
- **Database connection lost:** Circuit breaker, failover
- **High error rate:** Alert ops, disable feature

## Scaling Path

### Phase 1 (0-100 companies)
- Single NestJS instance
- Single PostgreSQL (Neon hobby tier)
- Single Redis (Upstash hobby tier)
- Per-company Fly.io Machines (auto-scale)
- Cost: ~$50-100/mo platform overhead

### Phase 2 (100-1K companies)
- 2-3 NestJS replicas (behind load balancer)
- Neon Pro (connection pooling, read replicas)
- Upstash Pro (higher throughput)
- Same per-company VMs
- Cost: ~$200-500/mo platform overhead

### Phase 3 (1K+ companies)
- 5+ NestJS replicas
- Database read replicas
- Redis cluster (Upstash)
- Advanced monitoring & auto-scaling
- Cost: ~$1000+/mo platform overhead

## Deployment Architecture

```
├── Control Plane (Railway/Fly.io)
│   ├── NestJS app (Docker container)
│   ├── Environment variables (secrets)
│   └── Health check endpoint (/api/internal/health)
│
├── Database (Neon/Supabase)
│   ├── PostgreSQL 15
│   ├── Connection pooling (PgBouncer)
│   └── Automated backups
│
├── Redis (Upstash)
│   ├── Pub/sub only (no persistence needed)
│   └── Per-request billing
│
├── S3 (AWS)
│   ├── Bucket for logs, attachments
│   └── Lifecycle rules (auto-delete old logs)
│
└── Per-Company VMs (Fly.io Machines)
    ├── One machine per company
    ├── Auto-hibernates when idle
    ├── Custom volumes for persistence
    └── Metrics reported back to control plane
```

## Error Handling Strategy

**Goal:** Fail gracefully. Always tell user what went wrong.

```
Domain Exception
    ↓
Caught by Handler
    ↓
Converted to HTTP response
    ↓
{
  "error": "Budget exceeded",
  "details": {
    "agentId": "...",
    "limit": 50000,
    "spent": 52000,
    "timestamp": "2026-03-17T10:30:00Z"
  }
}
    ↓
Sent to client (browser or agent callback)
```

**Patterns:**
- Specific domain exceptions for expected errors
- Generic 500 for unexpected errors (with correlation ID)
- All exceptions logged with context (user, company, request ID)
- User-friendly messages in UI

## Testing Strategy

- **Unit:** Handlers, services, utilities (Vitest, mocks)
- **Integration:** Repositories, database operations (test DB)
- **E2E:** Full user journeys (Playwright, real browser)
- **Performance:** Load testing (k6, k8s)

See [Testing Strategy](../blueprint/06-infrastructure/25-testing-strategy.md) for details.
