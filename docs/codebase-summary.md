# Codebase Summary

This document describes the **planned monorepo structure** for AI Company Platform v2. Since this is a greenfield project, all descriptions reflect the architecture blueprint; no code exists yet.

---

## Monorepo Overview

**Type:** pnpm workspaces + Turborepo
**Package scope:** @ai-company/*
**Language:** TypeScript (100%)
**Node version:** 18+ (runtime support for modern async/await, ESM)

```
ai-orchestration-company-v2/
├── apps/
│   ├── backend/          ← NestJS API server + Scheduler (CQRS, Clean Architecture)
│   ├── web/              ← React 19 frontend (Vite, React Router, React Query)
│   └── executor/         ← Agent Executor (Express, runs on Fly.io VMs)
├── packages/
│   ├── shared/           ← Types, constants, validators (monorepo shared code)
│   ├── adapters/         ← Agent runtime integrations (Claude, OpenClaw, Process)
│   └── adapter-utils/    ← Shared adapter utilities (process spawn, WebSocket)
├── config/
│   ├── skills/           ← Agent instruction templates (task protocol, skills)
│   └── templates/        ← Company templates (agent org charts, default configs)
├── tests/                ← E2E tests (Playwright)
├── .github/workflows/    ← CI/CD pipelines (GitHub Actions)
├── pnpm-workspace.yaml   ← Workspace definition
├── turbo.json            ← Turborepo config
└── package.json          ← Root package.json (monorepo root)
```

---

## Apps

### 1. apps/backend
**Type:** NestJS API server + task scheduler
**Port:** 3000 (dev), Railway/Fly.io (prod)
**Key dependencies:** @nestjs/core, @nestjs/schedule, drizzle-orm, zod, redis, ws

**Structure:**
```
backend/
├── src/
│   ├── presentation/         ← Controllers, DTOs, Guards, Interceptors, Filters
│   │   ├── controllers/
│   │   ├── dtos/
│   │   ├── guards/           ← JWT auth, API key validation
│   │   ├── interceptors/     ← Response transformation, logging
│   │   └── filters/          ← Global exception handling
│   ├── application/          ← Commands, Queries, Event Handlers, Application Services
│   │   ├── commands/
│   │   ├── queries/
│   │   ├── event-handlers/
│   │   ├── services/         ← ExecutionEngine, CostCalculator, etc.
│   │   └── dtos/             ← Internal DTO types (shared with domain)
│   ├── infrastructure/       ← Drizzle schemas, repositories, external clients
│   │   ├── database/
│   │   │   ├── schema.ts     ← All 35+ table definitions
│   │   │   ├── migrations/   ← DB version control
│   │   │   └── seed.ts       ← Initial data
│   │   ├── repositories/     ← Repository implementations
│   │   ├── clients/          ← External service clients (Fly API, S3, Redis)
│   │   └── config/           ← Environment, database config
│   ├── domain/               ← Entities, interfaces, enums, exceptions
│   │   ├── entities/
│   │   ├── repositories/     ← Repository interfaces (contracts)
│   │   ├── enums/
│   │   ├── exceptions/
│   │   └── types.ts          ← Domain types (zero framework imports)
│   ├── modules/              ← Feature modules (NestJS structure)
│   │   ├── shared.module.ts  ← Global module (auth, logging, config)
│   │   ├── api.module.ts     ← API routes (companies, agents, issues)
│   │   ├── scheduler.module.ts ← Heartbeat scheduling
│   │   └── realtime.module.ts  ← WebSocket, Redis pub/sub
│   ├── app.module.ts         ← Root app module
│   ├── main.ts               ← Bootstrap
│   └── utils/                ← Shared utilities
├── test/                     ← Unit tests (Vitest)
├── package.json
└── tsconfig.json
```

**Key Decisions:**
- **Clean Architecture:** 4 strict layers (no cross-layer circular dependencies)
- **CQRS:** Commands (async operations) separate from Queries (read-only)
- **Drizzle ORM:** Type-safe SQL builder, migrations in code
- **Zod:** Runtime validation for DTOs and API responses
- **Multi-tenant:** Every query filters by `companyId`

### 2. apps/web
**Type:** React 19 SPA
**Port:** 5173 (dev), CDN (prod)
**Key dependencies:** react, vite, react-router-dom, @tanstack/react-query, tailwindcss, shadcn/ui, ws

**Structure:**
```
web/
├── src/
│   ├── pages/                ← 11 pages (Auth, Onboarding, Dashboard, etc.)
│   │   ├── auth/
│   │   ├── onboarding/
│   │   ├── dashboard/
│   │   ├── team/
│   │   ├── agents/
│   │   ├── tasks/
│   │   ├── approvals/
│   │   ├── costs/
│   │   └── settings/
│   ├── components/           ← Reusable shadcn/ui + custom components
│   │   ├── ui/               ← shadcn/ui (Button, Input, Card, etc.)
│   │   ├── layout/           ← Header, Sidebar, Footer
│   │   ├── forms/            ← Form components (agent config, task creation)
│   │   ├── charts/           ← Cost trends, performance charts
│   │   └── realtime/         ← Live status, WebSocket consumers
│   ├── hooks/                ← React hooks (useCompany, useAgents, useWebSocket)
│   ├── services/             ← API clients (fetch wrappers)
│   ├── store/                ← State management (React Query, Context)
│   ├── types/                ← Shared types from @ai-company/shared
│   ├── utils/                ← Helpers (formatting, validation, etc.)
│   ├── App.tsx               ← Root component
│   └── main.tsx              ← React 19 render
├── public/                   ← Static assets
├── tests/                    ← Component tests (Vitest)
├── index.html                ← Entry HTML
├── vite.config.ts
├── tailwind.config.ts
├── package.json
└── tsconfig.json
```

**Key Decisions:**
- **React Router 7:** Client-side routing with loaders, actions
- **React Query:** Server state management, caching, syncing
- **Tailwind CSS 4:** Utility-first styling
- **shadcn/ui:** Unstyled, composable component library
- **WebSocket:** Real-time updates (agent status, task progress, costs)

### 3. apps/executor
**Type:** Express.js agent executor service (Fly.io VM)
**Port:** 8080 (both dev & production)
**Key dependencies:** express, zod, node-process-utils, ws, redis

**Purpose:** Receives execution requests from Control Plane API → spawns adapter processes (Claude CLI, OpenClaw, etc.) → streams results back via SSE or WebSocket.

**Structure:**
```
executor/
├── src/
│   ├── handlers/            ← Request handlers (execute-task, get-status, etc.)
│   ├── adapters/            ← Adapter implementations (index, Claude, OpenClaw, Process)
│   │   ├── index.ts         ← AdapterRegistry
│   │   ├── claude-cli.ts
│   │   ├── openclaw-websocket.ts
│   │   ├── process-generic.ts
│   │   └── types.ts         ← IAdapter interface
│   ├── workspace/           ← Git repo, file management
│   ├── cost-tracker/        ← Token + compute cost calculation
│   ├── server.ts            ← Express setup
│   ├── routes/
│   │   ├── execute.ts       ← POST /execute (trigger work)
│   │   ├── status.ts        ← GET /status (agent status)
│   │   └── sse.ts           ← SSE stream handler
│   ├── app.ts               ← App initialization
│   └── utils/               ← Shared utilities
├── tests/
├── Dockerfile               ← Container for Fly.io deployment
├── package.json
└── tsconfig.json
```

**Key Decisions:**
- **Express (not NestJS):** Lightweight, minimal overhead for specialized executor service
- **Adapter Registry:** Dynamic loading of adapter implementations
- **SSE + WebSocket:** Dual streaming support (API ↔ Client)
- **Workspace:** Persistent filesystem per company VM (git repos, agent memory)

---

## Packages

### 1. packages/shared
**Type:** Shared types, constants, validators
**Exports:** Types, enums, Zod schemas, constants

**Structure:**
```
shared/
├── src/
│   ├── types/
│   │   ├── company.ts       ← Company, Agent, Issue types
│   │   ├── execution.ts     ← Heartbeat, Run, Event types
│   │   ├── cost.ts          ← Cost, Budget types
│   │   └── api.ts           ← API request/response types
│   ├── schemas/             ← Zod validators
│   │   ├── company.schema.ts
│   │   ├── agent.schema.ts
│   │   ├── issue.schema.ts
│   │   └── execution.schema.ts
│   ├── constants/
│   │   ├── roles.ts         ← Agent roles (CEO, CTO, Engineer, etc.)
│   │   ├── statuses.ts      ← Task statuses, run states
│   │   ├── permissions.ts   ← Agent permissions (canHire, canApprove, etc.)
│   │   └── config.ts        ← Default values
│   ├── enums/
│   │   ├── adapter-type.enum.ts
│   │   ├── issue-status.enum.ts
│   │   └── heartbeat-state.enum.ts
│   └── index.ts             ← Barrel export
├── package.json
└── tsconfig.json
```

**Usage:** Imported by backend, web, executor, adapter-utils.

### 2. packages/adapters
**Type:** Agent runtime integrations (V1: 3 of 9)
**Exports:** Adapter implementations, types

**V1 Adapters:**
1. **Claude CLI** — `@ai-company/adapters/claude-cli`
2. **OpenClaw WebSocket** — `@ai-company/adapters/openclaw-websocket`
3. **Process (generic)** — `@ai-company/adapters/process-generic`

**Structure:**
```
adapters/
├── src/
│   ├── index.ts             ← AdapterRegistry singleton
│   ├── types.ts             ← IAdapter interface, ExecutionRequest/Response
│   ├── claude-cli/
│   │   ├── index.ts
│   │   ├── spawner.ts
│   │   ├── prompt-builder.ts
│   │   └── cost-calculator.ts
│   ├── openclaw-websocket/
│   │   ├── index.ts
│   │   ├── websocket-client.ts
│   │   ├── message-handler.ts
│   │   └── cost-calculator.ts
│   ├── process-generic/
│   │   ├── index.ts
│   │   ├── subprocess.ts
│   │   └── env-builder.ts
│   ├── errors/              ← AdapterError, TimeoutError, etc.
│   └── utils/               ← Shared adapter utilities (retry, logging)
├── tests/
├── package.json
└── tsconfig.json
```

**Extension path:** Future adapters (OpenAI, Gemini, Cursor, etc.) added as new folders.

### 3. packages/adapter-utils
**Type:** Shared utilities for all adapters
**Exports:** Process utilities, WebSocket helpers, retry logic

**Structure:**
```
adapter-utils/
├── src/
│   ├── process/
│   │   ├── spawn-safe.ts   ← Spawn process with timeout, resource limits
│   │   ├── stream-handler.ts ← Capture stdout/stderr
│   │   └── signal-handler.ts ← SIGTERM, SIGKILL handling
│   ├── websocket/
│   │   ├── reconnect.ts    ← Auto-reconnect with exponential backoff
│   │   ├── message-queue.ts ← Buffer messages during disconnects
│   │   └── heartbeat.ts    ← Ping/pong keep-alive
│   ├── retry/
│   │   └── exponential-backoff.ts ← Retry with exponential backoff
│   ├── cost/
│   │   ├── token-counter.ts ← Count tokens from API responses
│   │   └── compute-calculator.ts ← VM execution cost
│   └── index.ts             ← Barrel export
├── tests/
├── package.json
└── tsconfig.json
```

**Usage:** Imported by `packages/adapters` and `apps/executor`.

---

## Config Directory

### config/skills/
Agent instruction templates. Each file is a prompt that defines agent behavior/capabilities at runtime.

```
skills/
├── shared/
│   ├── task-protocol.md    ← How agents checkout/complete tasks
│   ├── permissions.md      ← What agents can/cannot do
│   └── tool-definitions.md ← Tool signatures (code execution, file access, etc.)
├── roles/
│   ├── ceo.md
│   ├── cto.md
│   ├── engineer.md
│   ├── designer.md
│   └── marketer.md
└── templates/
    ├── default-org-chart.json ← 5-agent starter team
    └── custom-org-chart.json  ← Larger team template
```

### config/templates/
Company scaffolding templates.

```
templates/
├── companies/
│   ├── saas-startup.json   ← Default: CEO + CTO + 2 engineers + designer + marketer
│   ├── agency.json         ← Creative focus: PM + 3 designers + 2 marketers
│   └── custom.json         ← User uploads custom JSON
└── agents/
    ├── agent-configs/      ← Default agent configs (budgets, permissions)
    └── cost-models/        ← Pricing per adapter type
```

---

## Database Schema (Planned)

**Technology:** Drizzle ORM + PostgreSQL (Neon)
**Tables:** 35+
**Multi-tenant:** Every table includes `companyId`

### Core Tables
| Table | Purpose |
|-------|---------|
| `users` | Platform users (email, hashed password, created_at) |
| `companies` | Org units (name, goal, budget, status) |
| `company_api_keys` | LLM API keys (encrypted, per-company) |
| `company_vms` | Fly.io VM details (VM ID, status, hibernation) |
| `invites` | Company invitations (email, token, expires_at) |

### Team Tables
| Table | Purpose |
|-------|---------|
| `agents` | Agent definitions (name, role, adapter_type, budget) |
| `agent_permissions` | Granular permissions (canHire, canApprove, canTerminate) |
| `agent_cost_limits` | Budget enforcement (monthly_limit, alert_threshold) |
| `agent_hierarchy` | Org chart (manager_id, direct_reports) |

### Work Tables
| Table | Purpose |
|-------|---------|
| `issues` | Tasks (title, description, status, assigned_to) |
| `issue_history` | Status transitions (issue_id, old_status, new_status) |
| `issue_comments` | Comments & activity (issue_id, author, text) |
| `issue_attachments` | Files (issue_id, url, s3_key) |

### Execution Tables
| Table | Purpose |
|-------|---------|
| `heartbeat_schedules` | Recurring schedules (agent_id, interval, enabled) |
| `heartbeat_runs` | Execution instances (schedule_id, status, started_at, ended_at) |
| `heartbeat_run_events` | Execution trace (run_id, event_type, data) |

### Cost Tables
| Table | Purpose |
|-------|---------|
| `cost_events` | Token + compute costs (run_id, token_count, compute_cost) |
| `budget_alerts` | Spending warnings (agent_id, alerting_at, threshold_percent) |
| `monthly_billing` | Invoices (company_id, month, total_cost, status) |

### Activity & Governance
| Table | Purpose |
|-------|---------|
| `activity_log` | All actions (user_id, action, target_id, created_at) |
| `approvals` | Pending decisions (requester_id, type, data, status) |
| `audit_events` | Security audit trail (actor, resource, action, timestamp) |

**Total:** ~35 tables. Specific migration files generated during Phase 3 (Database Schema).

---

## Key File Naming Conventions

### Backend (NestJS)
| Artifact | Pattern | Example |
|----------|---------|---------|
| Controller | `{name}.controller.ts` | `company.controller.ts` |
| Service | `{name}.service.ts` | `execution-engine.service.ts` |
| Module | `{name}.module.ts` | `api.module.ts` |
| Guard | `{name}.guard.ts` | `jwt-auth.guard.ts` |
| Interceptor | `{name}.interceptor.ts` | `logging.interceptor.ts` |
| Filter | `{name}.filter.ts` | `global-exception.filter.ts` |
| DTO | `{name}.dto.ts` | `create-company.dto.ts` |
| Entity | `{name}.entity.ts` | `company.entity.ts` |
| Repository | `{name}.repository.ts` | `company.repository.ts` |
| Command | `{name}.command.ts` | `create-company.command.ts` |
| Query | `{name}.query.ts` | `get-agents.query.ts` |
| Event | `{name}.event.ts` | `company-created.event.ts` |
| Exception | `{name}.exception.ts` | `company-not-found.exception.ts` |

### Frontend (React)
| Artifact | Pattern | Example |
|----------|---------|---------|
| Component | `{Name}.tsx` (PascalCase) | `Dashboard.tsx` |
| Page | `{Name}.tsx` | `DashboardPage.tsx` |
| Hook | `use{Name}.ts` | `useCompany.ts` |
| Service | `{name}.service.ts` | `api.service.ts` |
| Type | `{name}.types.ts` | `company.types.ts` |
| Constant | `{name}.constant.ts` | `roles.constant.ts` |
| Utility | `{name}.util.ts` | `format-date.util.ts` |

### Shared (packages/shared)
| Artifact | Pattern | Example |
|----------|---------|---------|
| Type | `{name}.ts` | `company.ts` |
| Schema | `{name}.schema.ts` | `company.schema.ts` |
| Enum | `{name}.enum.ts` | `role.enum.ts` |
| Constant | `{name}.constant.ts` | `permissions.constant.ts` |

---

## Dependencies Summary

### Backend Stack
```json
{
  "@nestjs/core": "^10.x",
  "@nestjs/schedule": "^4.x",
  "drizzle-orm": "^0.30.x",
  "pg": "^8.x",
  "zod": "^3.x",
  "redis": "^4.x",
  "ws": "^8.x",
  "@better-auth/core": "^0.x"
}
```

### Frontend Stack
```json
{
  "react": "^19.x",
  "react-router-dom": "^7.x",
  "@tanstack/react-query": "^5.x",
  "vite": "^5.x",
  "tailwindcss": "^4.x",
  "@shadcn/ui": "^0.x"
}
```

### Shared & Executor
```json
{
  "zod": "^3.x",
  "typescript": "^5.x"
}
```

---

## Module Organization

### Backend Modules (NestJS)
1. **SharedModule** (global) — Auth, logging, config, env validation
2. **ApiModule** — Companies, agents, issues, costs, approvals
3. **SchedulerModule** — Heartbeat scheduling, execution dispatch
4. **RealtimeModule** — WebSocket, Redis pub/sub, live updates

### Feature Domains
Each domain encapsulates:
- **Controllers** (HTTP endpoints)
- **Services** (business logic)
- **Repositories** (data access)
- **Commands/Queries** (async handlers)
- **Domain logic** (entities, enums, exceptions)
- **DTOs** (input/output contracts)

### Import Layers (Enforced via ESLint)
```
presentation → application → infrastructure ↔ domain
Domain never imports from other layers
Infrastructure never imports from presentation/application
```

---

## Code Quality Standards

### TypeScript
- Strict mode enabled (`"strict": true`)
- No implicit any
- All async operations typed with `Promise<T>`
- Shared types centralized in `packages/shared`

### Naming
- **Files:** kebab-case (e.g., `execution-engine.service.ts`)
- **Classes/Interfaces:** PascalCase (e.g., `ExecutionEngine`, `IRepository`)
- **Variables/Functions:** camelCase (e.g., `executeTask()`, `agentId`)
- **Constants:** UPPER_SNAKE_CASE (e.g., `MAX_TOKEN_LIMIT`)
- **DB Columns:** snake_case (e.g., `created_at`, `company_id`)

### File Size
- **Max LOC per file:** 200 lines (strict)
- **Classes:** Single responsibility, <50 methods
- **Functions:** <30 lines average, pure when possible

### Testing
- **Unit tests:** Vitest, >80% coverage for critical paths
- **Integration tests:** Database + repository layer
- **E2E tests:** Playwright for user journeys (UI + API)

---

## Development Workflow

### Package Scripts
```bash
# Root
pnpm install              # Install all workspaces
pnpm build                # Build all packages (parallel via Turborepo)
pnpm test                 # Run all tests
pnpm dev                  # Start all dev servers (backend, web, executor)
pnpm lint                 # TypeScript + ESLint checks

# Individual workspaces
pnpm -F @ai-company/adapters build
pnpm -F @ai-company/web dev
```

### CI/CD Pipeline
1. **Typecheck:** `tsc --noEmit`
2. **Unit tests:** `vitest run --coverage`
3. **Build:** `pnpm build` (all packages)
4. **E2E tests:** `playwright test` (if UI changed)
5. **Deploy:** Railway (API), Vercel (frontend), Fly.io (executor)

---

## Directory Structure Reference

```
ai-orchestration-company-v2/
├── apps/backend/src/           ← NestJS API (CQRS, Clean Architecture)
├── apps/web/src/               ← React 19 SPA (11 pages)
├── apps/executor/src/          ← Express executor (Fly.io)
├── packages/shared/src/        ← Types, validators, enums
├── packages/adapters/src/      ← Adapter implementations
├── packages/adapter-utils/src/ ← Shared adapter utilities
├── config/skills/              ← Agent instruction templates
├── config/templates/           ← Company/agent templates
├── tests/                      ← E2E tests (Playwright)
├── docs/                       ← Operational documentation
├── docs/blueprint/             ← Detailed specs (not modified)
├── plans/                      ← Implementation phases
└── .github/workflows/          ← CI/CD pipelines
```

---

**Version:** 1.0
**Last Updated:** 2026-03-17
**Status:** Planned structure (no code yet)
**See Also:** `./docs/blueprint/03-architecture/14-monorepo-setup-guide.md` for detailed setup instructions.
