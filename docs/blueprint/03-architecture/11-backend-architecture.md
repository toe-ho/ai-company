# 11 — Backend Architecture

High-level overview of the NestJS API server. For full directory structure and code examples, see [12 — API Architecture (NestJS + Drizzle ORM + CQRS)](12-api-architecture-nestjs.md).

## Framework & Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Node.js 20+ |
| Framework | NestJS |
| Language | TypeScript |
| ORM | Drizzle ORM |
| Auth | Better Auth (users) + JWT/API key (agents) |
| Validation | Zod (via ZodValidationPipe) |
| Real-time | Redis pub/sub (Upstash) + WebSocket Gateway |
| Logging | Pino |

## Clean Architecture Layers

The API is structured in four strict layers. Inner layers never import from outer layers.

```
┌──────────────────────────────────────────────────────────┐
│  Presentation Layer                                       │
│  Controllers, DTOs, Guards, Interceptors, Filters        │
├──────────────────────────────────────────────────────────┤
│  Application Layer                                        │
│  Commands, Queries, Event Handlers, Services             │
├──────────────────────────────────────────────────────────┤
│  Infrastructure Layer                                     │
│  Drizzle Schemas, Repositories, External Clients         │
├──────────────────────────────────────────────────────────┤
│  Domain Layer  (zero framework imports)                   │
│  Entities, Repository Interfaces, Enums, Exceptions      │
└──────────────────────────────────────────────────────────┘
```

### Domain Layer
Pure TypeScript — no NestJS, no Drizzle. Defines:
- Entity interfaces (Company, Agent, Issue, HeartbeatRun, etc.)
- Repository contracts (`ICompanyRepository`, `IIssueRepository`, etc.)
- Enums (AgentStatus, IssueStatus, RunStatus, etc.)
- Domain exceptions (`IssueAlreadyCheckedOutException`, `AgentOverBudgetException`, etc.)

### Application Layer
Use cases implemented as CQRS handlers. Also contains:
- Cross-cutting service interfaces (`IProvisionerService`, `IApiKeyVaultService`, etc.)
- Domain event definitions and handlers
- Actor context resolution

### Infrastructure Layer
Concrete implementations:
- Drizzle table schemas (mapped to DB tables)
- Repository implementations (Drizzle queries)
- External service clients: Fly.io, Redis, S3, Stripe

### Presentation Layer
HTTP entry points:
- Controllers dispatch to `commandBus` or `queryBus` — no business logic
- DTOs validated by `ZodValidationPipe`
- Guards, decorators, and interceptors handle cross-cutting HTTP concerns

## CQRS Pattern

All state mutations go through **Commands**; all reads go through **Queries**. This enforces a clean read/write separation.

```
Write path:
  Controller → commandBus.execute(new XyzCommand(...)) → XyzHandler → Repository

Read path:
  Controller → queryBus.execute(new XyzQuery(...)) → XyzHandler → Repository → DTO
```

**Commands** (examples):
- `CreateCompanyCommand` / `CreateCompanyFromTemplateCommand`
- `CheckoutIssueCommand` — atomic, returns 409 on conflict
- `InvokeHeartbeatCommand` — the big orchestrator
- `EnsureVmCommand` / `HibernateVmCommand`
- `StoreApiKeyCommand` / `ValidateApiKeyCommand`

**Queries** (examples):
- `ListAgentsQuery` / `GetAgentQuery`
- `GetHeartbeatContextQuery` — assembles full context for agent execution
- `GetDashboardSummaryQuery`
- `ListRunsQuery` / `GetRunQuery`

## Module Structure

NestJS modules group related providers. Key modules:

| Module | Purpose |
|--------|---------|
| `SharedModule` | Global — all repositories, services, command/query/event handlers |
| `ApiModule` | HTTP controllers (board, agent, internal, public routes) |
| `SchedulerModule` | Heartbeat timer + cron jobs (@nestjs/schedule, pg advisory lock) |
| `RealtimeModule` | WebSocket gateway + Redis pub/sub bridge |

The `SharedModule` is `@Global()` so repositories and services are available everywhere without re-importing.

## Service Layer

Application services implement cross-cutting concerns behind interfaces. Handlers call services through interfaces, enabling easy testing with mocks.

| Interface | Implementation | Purpose |
|-----------|---------------|---------|
| `IExecutionEngineService` | `ExecutionEngineService` | HTTP POST to Fly.io VM + SSE parse |
| `IProvisionerService` | `FlyioProvisionerService` | Fly.io Machines API lifecycle |
| `IApiKeyVaultService` | `ApiKeyVaultService` | AES-256 encrypt/decrypt/validate |
| `ILiveEventsService` | `RedisLiveEventsService` | Redis PUBLISH/SUBSCRIBE |
| `IStorageService` | `S3StorageService` | S3 put/get/delete |
| `IEncryptionService` | `AesEncryptionService` | AES-256 primitives |

## Heartbeat Service (Core Orchestrator)

`InvokeHeartbeatHandler` is the most critical handler. It orchestrates:

1. Validate agent is active and under budget
2. Retrieve company's LLM API key via `IApiKeyVaultService`
3. Boot or wake the Fly.io VM via `EnsureVmCommand`
4. Build `ExecutionRequest` (context, session, API key, agent JWT)
5. Call `IExecutionEngineService.execute(request)` — returns SSE stream
6. For each SSE event: persist `HeartbeatRunEvent`, publish live event via `ILiveEventsService`
7. On completion: record cost event, update `AgentRuntimeState`, check budget
8. Schedule idle VM hibernate after 10 minutes

The scheduler runs as a **built-in module** within `apps/backend/` using `@nestjs/schedule`. It acquires a **pg advisory lock** to prevent duplicate execution when multiple API replicas are deployed. Every 30 seconds it runs:
1. `ReapOrphanedRunsCommand` — clean stale runs (>5 min without update)
2. Resume queued runs
3. `tickTimers` — enqueue runs for agents whose heartbeat interval has elapsed

## Provisioner Service (Fly.io)

Manages per-company Fly.io VM lifecycle:

```
stopped → starting → running → hibernating → stopped
```

- `EnsureVmCommand` boots or wakes the VM (~3s cold start)
- VM auto-hibernates after configurable idle timeout (default 10 min)
- `DestroyVmCommand` for permanent teardown (agent terminated)
- State tracked in `companyVms` table

## API Key Vault Service

Stores user's LLM API keys encrypted at rest:

```typescript
interface IApiKeyVaultService {
  store(companyId, provider, key): Promise<void>;   // AES-256 encrypt + save
  retrieve(companyId, provider): Promise<string>;   // Decrypt for injection
  validate(companyId, provider): Promise<boolean>;  // Test against provider
  revoke(companyId, provider): Promise<void>;
}
```

Keys encrypted with AES-256. Never stored on VMs. Injected as environment variables during execution only.

## Guards, Decorators, and Interceptors

NestJS cross-cutting concerns applied at the controller level:

**Guards** (authentication + authorization):
- `BoardAuthGuard` — validates session cookie (Better Auth)
- `AgentAuthGuard` — validates agent JWT or hashed API key
- `CompanyAccessGuard` — verifies actor belongs to the requested company
- `CompanyRoleGuard` — enforces owner/admin/viewer roles

**Decorators** (parameter extraction):
- `@CurrentActor()` → resolves `IActor` from request context
- `@CompanyId()` → extracts company UUID from route params or actor
- `@RunId()` → extracts `X-Run-Id` header (links agent actions to a heartbeat run)
- `@Roles('owner', 'admin')` → declares required roles

**Interceptors** (cross-cutting behavior):
- `ActivityLogInterceptor` — auto-logs mutations to the activity log
- `CompanyScopeInterceptor` — auto-injects `companyId` into queries
- `HttpLoggerInterceptor` — structured request/response logging

**Filter:**
- `HttpExceptionFilter` — formats all errors as `{ error: "message", details: { ... } }`

**Pipe:**
- `ZodValidationPipe` — validates request DTOs against Zod schemas; returns 400 with field-level errors on failure

## Scheduler (Built-in Module)

The heartbeat scheduler runs as `SchedulerModule` within `apps/backend/` using `@nestjs/schedule`. It acquires a **PostgreSQL advisory lock** on each tick so only one replica runs the tick loop at a time, making it safe in multi-replica deployments.

Tick interval: 30 seconds (`@Interval(30000)`).

## Controller Routing

Controllers are split by auth surface:

| Group | Prefix | Auth | Purpose |
|-------|--------|------|---------|
| `board/` | `/api/...` | Session cookie | Human dashboard users |
| `agent/` | `/api/...` | JWT or API key | Agent callback endpoints |
| `public/` | `/api/...` | None | Login, signup, template browsing |
| `internal/` | `/api/...` | None (internal only) | Health check |

See [23 — API Architecture](23-api-architecture-nestjs.md) for the full directory structure, Drizzle schema example, CQRS handler example, and module registration code.
