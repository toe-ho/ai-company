# Codebase Summary

**Status:** Blueprint phase. Source code structure planned but not yet implemented.

This document describes the planned monorepo structure, packages, and dependency relationships based on the technical blueprints.

## Monorepo Layout

### Root Structure
```
project-root/
├── apps/                      ← Application packages
├── packages/                  ← Shared libraries
├── config/                    ← Configuration files
├── tests/                     ← End-to-end tests
├── turbo.json                 ← Turborepo config
├── package.json               ← Root pnpm workspaces
├── pnpm-workspace.yaml        ← Workspace definition
├── .github/workflows/         ← GitHub Actions CI/CD
├── .env.example               ← Example environment variables
└── README.md                  ← Project root documentation
```

## Application Packages

### apps/backend
NestJS-based control plane API server and heartbeat scheduler.

**Purpose:** Orchestrate agent execution, manage auth, track costs, schedule heartbeats

**Key Directories:**
```
apps/backend/
├── src/
│   ├── main.ts                     ← App bootstrap
│   ├── app.module.ts               ← Root module
│   │
│   ├── domain/                     ← Pure TypeScript (no framework)
│   │   ├── entities/               ← Company, Agent, Issue entities
│   │   ├── repositories/           ← ICompanyRepository, IAgentRepository, etc.
│   │   ├── enums/                  ← AgentStatus, IssueStatus, RunStatus
│   │   ├── exceptions/             ← BudgetExceededException, etc.
│   │   └── value-objects/          ← Money, Cost, Uuid, etc.
│   │
│   ├── infrastructure/             ← Database & external services
│   │   ├── database/
│   │   │   ├── schemas/            ← Drizzle table definitions
│   │   │   │   ├── companies.ts
│   │   │   │   ├── agents.ts
│   │   │   │   ├── issues.ts
│   │   │   │   ├── heartbeat-runs.ts
│   │   │   │   ├── heartbeat-run-events.ts
│   │   │   │   ├── cost-events.ts
│   │   │   │   ├── company-api-keys.ts
│   │   │   │   ├── activity-log.ts
│   │   │   │   └── migrations/     ← Drizzle migrations
│   │   │   └── db.ts               ← Drizzle client
│   │   │
│   │   ├── mappers/                ← Row → Entity mappers
│   │   │   ├── company.mapper.ts
│   │   │   ├── agent.mapper.ts
│   │   │   └── ...
│   │   │
│   │   ├── repositories/           ← Repository implementations
│   │   │   ├── company.repository.ts
│   │   │   ├── agent.repository.ts
│   │   │   ├── issue.repository.ts
│   │   │   └── ...
│   │   │
│   │   ├── services/               ← External service clients
│   │   │   ├── execution-engine.service.ts    ← HTTP to Fly.io VM
│   │   │   ├── provisioner.service.ts         ← Fly.io lifecycle
│   │   │   ├── api-key-vault.service.ts       ← AES-256 encryption
│   │   │   ├── live-events.service.ts         ← Redis pub/sub
│   │   │   ├── storage.service.ts             ← S3 operations
│   │   │   └── encryption.service.ts          ← AES-256 primitives
│   │   │
│   │   └── infrastructure.module.ts
│   │
│   ├── application/                ← CQRS handlers, use cases
│   │   ├── commands/               ← Command handlers
│   │   │   ├── create-company/
│   │   │   │   ├── create-company.command.ts
│   │   │   │   ├── create-company.handler.ts
│   │   │   │   └── create-company.dto.ts
│   │   │   ├── invoke-heartbeat/
│   │   │   ├── ensure-vm/
│   │   │   ├── checkout-issue/
│   │   │   ├── create-agent/
│   │   │   └── ... (50+ commands)
│   │   │
│   │   ├── queries/                ← Query handlers
│   │   │   ├── get-agent/
│   │   │   │   ├── get-agent.query.ts
│   │   │   │   ├── get-agent.handler.ts
│   │   │   │   └── get-agent.dto.ts
│   │   │   ├── list-agents/
│   │   │   ├── get-heartbeat-context/
│   │   │   └── ... (30+ queries)
│   │   │
│   │   ├── events/                 ← Domain event handlers
│   │   │   ├── company-created/
│   │   │   ├── heartbeat-completed/
│   │   │   ├── budget-exceeded/
│   │   │   └── ...
│   │   │
│   │   └── application.module.ts
│   │
│   ├── api/                        ← HTTP entry points
│   │   ├── controllers/
│   │   │   ├── board/              ← User dashboard endpoints
│   │   │   │   ├── companies.controller.ts
│   │   │   │   ├── agents.controller.ts
│   │   │   │   ├── issues.controller.ts
│   │   │   │   ├── heartbeats.controller.ts
│   │   │   │   └── billing.controller.ts
│   │   │   │
│   │   │   ├── agent/              ← Agent callback endpoints
│   │   │   │   ├── issues.controller.ts
│   │   │   │   ├── status.controller.ts
│   │   │   │   └── execution.controller.ts
│   │   │   │
│   │   │   ├── public/             ← Public endpoints (no auth)
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── templates.controller.ts
│   │   │   │   └── health.controller.ts
│   │   │   │
│   │   │   └── internal/           ← Internal endpoints
│   │   │       └── health.controller.ts
│   │   │
│   │   ├── dtos/                   ← Request/response DTOs
│   │   │   ├── create-company.dto.ts
│   │   │   ├── get-company.dto.ts
│   │   │   ├── create-agent.dto.ts
│   │   │   └── ... (50+ DTOs)
│   │   │
│   │   ├── guards/                 ← Authentication & authorization
│   │   │   ├── board-auth.guard.ts
│   │   │   ├── agent-auth.guard.ts
│   │   │   ├── company-access.guard.ts
│   │   │   └── company-role.guard.ts
│   │   │
│   │   ├── decorators/             ← Parameter extraction
│   │   │   ├── current-actor.decorator.ts
│   │   │   ├── company-id.decorator.ts
│   │   │   ├── run-id.decorator.ts
│   │   │   ├── roles.decorator.ts
│   │   │   └── public.decorator.ts
│   │   │
│   │   ├── interceptors/           ← Cross-cutting HTTP behavior
│   │   │   ├── activity-log.interceptor.ts
│   │   │   ├── company-scope.interceptor.ts
│   │   │   └── http-logger.interceptor.ts
│   │   │
│   │   ├── filters/                ← Exception handling
│   │   │   └── http-exception.filter.ts
│   │   │
│   │   ├── pipes/                  ← Input validation
│   │   │   └── zod-validation.pipe.ts
│   │   │
│   │   └── api.module.ts
│   │
│   ├── scheduler/                  ← Heartbeat scheduling
│   │   ├── heartbeat.scheduler.ts
│   │   ├── scheduled-events.ts     ← Event handlers for scheduler
│   │   └── scheduler.module.ts
│   │
│   ├── realtime/                   ← WebSocket & real-time events
│   │   ├── websocket.gateway.ts
│   │   ├── realtime.service.ts
│   │   └── realtime.module.ts
│   │
│   ├── shared/                     ← @Global() module
│   │   ├── shared.module.ts        ← Registers all repos, handlers, services
│   │   └── index.ts                ← Re-exports
│   │
│   └── config/                     ← Configuration
│       └── env.ts                  ← Environment variables
│
├── test/
│   ├── unit/                       ← Unit tests (Vitest)
│   │   ├── handlers/
│   │   ├── services/
│   │   └── repositories/
│   │
│   └── integration/                ← Integration tests (Drizzle + test DB)
│       ├── repositories/
│       └── handlers/
│
├── Dockerfile
├── package.json
├── tsconfig.json
└── jest.config.js
```

**Dependencies:**
```json
{
  "@nestjs/common": "^10.0.0",
  "@nestjs/core": "^10.0.0",
  "@nestjs/cqrs": "^10.0.0",
  "@nestjs/schedule": "^4.0.0",
  "@nestjs/websockets": "^10.0.0",
  "drizzle-orm": "^0.30.0",
  "drizzle-kit": "^0.21.0",
  "zod": "^3.22.0",
  "pino": "^8.17.0",
  "better-auth": "latest",
  "aws-sdk": "^2.0.0",
  "@upstash/redis": "^1.25.0",
  "postgres": "^3.4.0"
}
```

### apps/web
React frontend dashboard (Vite, Tailwind, shadcn/ui).

**Purpose:** User interface for company management, team building, task tracking

**Key Directories:**
```
apps/web/
├── src/
│   ├── main.tsx                    ← React app bootstrap
│   ├── App.tsx                     ← Root component
│   │
│   ├── pages/                      ← Route pages
│   │   ├── dashboard/
│   │   │   ├── index.tsx           ← Company overview
│   │   │   ├── agents.tsx          ← Agent management
│   │   │   ├── tasks.tsx           ← Task board
│   │   │   ├── org-chart.tsx       ← Organization visualization
│   │   │   ├── approvals.tsx       ← Approval workflows
│   │   │   └── billing.tsx         ← Cost tracking
│   │   │
│   │   ├── auth/
│   │   │   ├── login.tsx
│   │   │   ├── signup.tsx
│   │   │   └── reset-password.tsx
│   │   │
│   │   └── settings/
│   │       ├── profile.tsx
│   │       └── company.tsx
│   │
│   ├── components/                 ← Reusable UI components
│   │   ├── dashboard/
│   │   │   ├── company-overview.tsx
│   │   │   ├── agent-card.tsx
│   │   │   ├── task-list.tsx
│   │   │   └── org-chart.tsx
│   │   │
│   │   ├── common/
│   │   │   ├── header.tsx
│   │   │   ├── sidebar.tsx
│   │   │   ├── loading.tsx
│   │   │   └── error-boundary.tsx
│   │   │
│   │   └── forms/
│   │       ├── create-company.tsx
│   │       ├── create-agent.tsx
│   │       └── create-task.tsx
│   │
│   ├── hooks/                      ← Custom React hooks
│   │   ├── use-company.ts          ← Fetch company data
│   │   ├── use-agents.ts           ← Fetch agents
│   │   ├── use-tasks.ts            ← Fetch tasks
│   │   ├── use-websocket.ts        ← Real-time connection
│   │   └── use-auth.ts             ← Auth state
│   │
│   ├── stores/                     ← State management
│   │   ├── auth.store.ts           ← Auth state
│   │   ├── company.store.ts        ← Selected company
│   │   ├── ui.store.ts             ← UI state (modals, etc.)
│   │   └── realtime.store.ts       ← Live event state
│   │
│   ├── api/                        ← API client
│   │   ├── client.ts               ← Fetch wrapper
│   │   ├── companies.api.ts        ← Company endpoints
│   │   ├── agents.api.ts           ← Agent endpoints
│   │   ├── tasks.api.ts            ← Task endpoints
│   │   └── heartbeats.api.ts       ← Heartbeat endpoints
│   │
│   ├── types/                      ← TypeScript types
│   │   ├── company.ts
│   │   ├── agent.ts
│   │   ├── task.ts
│   │   └── api.ts
│   │
│   ├── utils/                      ← Utility functions
│   │   ├── format.ts
│   │   ├── validation.ts
│   │   └── api-client.ts
│   │
│   ├── styles/                     ← Global styles
│   │   └── globals.css
│   │
│   └── config/
│       └── env.ts                  ← Environment variables
│
├── public/
│   └── assets/
│
├── index.html
├── vite.config.ts
├── tsconfig.json
├── package.json
└── tailwind.config.js
```

**Dependencies:**
```json
{
  "react": "^18.2.0",
  "react-dom": "^18.2.0",
  "react-router-dom": "^6.20.0",
  "zustand": "^4.4.0",
  "@tanstack/react-query": "^5.28.0",
  "tailwindcss": "^3.4.0",
  "shadcn-ui": "latest",
  "axios": "^1.6.0",
  "zod": "^3.22.0"
}
```

### apps/executor
Agent Executor running on Fly.io VM. Receives execution requests and manages agent processes.

**Purpose:** Run AI agent processes in isolated per-company VM

**Key Directories:**
```
apps/executor/
├── src/
│   ├── main.ts                     ← HTTP server bootstrap
│   ├── executor/
│   │   ├── executor.ts             ← Core executor class
│   │   ├── execution-request.ts    ← Request type
│   │   ├── execution-event.ts      ← Event types (stdout, stderr, done)
│   │   └── sse-stream.ts           ← SSE response streaming
│   │
│   ├── adapters/                   ← Local adapter implementations
│   │   ├── claude.adapter.ts       ← Claude CLI wrapper
│   │   ├── codex.adapter.ts        ← Codex CLI wrapper
│   │   ├── gemini.adapter.ts       ← Gemini API wrapper
│   │   ├── opencode.adapter.ts     ← OpenCode CLI wrapper
│   │   ├── openclaw-gateway.adapter.ts
│   │   ├── cursor.adapter.ts
│   │   ├── process.adapter.ts      ← Process executor
│   │   ├── http.adapter.ts         ← HTTP client
│   │   ├── pi.adapter.ts           ← Pi CLI wrapper
│   │   └── adapter-factory.ts      ← Registry factory
│   │
│   ├── workspace/                  ← File system management
│   │   ├── workspace.ts            ← Root workspace
│   │   ├── git-manager.ts          ← Git operations
│   │   ├── file-manager.ts         ← File read/write
│   │   └── persistence.ts          ← Volume mounting
│   │
│   ├── session/
│   │   ├── session-codec.ts        ← Serialize/deserialize agent state
│   │   └── session-storage.ts      ← Persistent session store
│   │
│   ├── config/
│   │   └── env.ts                  ← Environment variables
│   │
│   └── utils/
│       ├── stream-parser.ts        ← Parse adapter output
│       ├── error-handler.ts        ← Graceful error handling
│       └── logging.ts              ← Request logging
│
├── test/
│   └── unit/
│       ├── adapters/
│       └── workspace/
│
├── Dockerfile
├── package.json
└── tsconfig.json
```

**Dependencies:**
```json
{
  "express": "^4.18.0",
  "@upstash/redis": "^1.25.0",
  "zod": "^3.22.0",
  "pino": "^8.17.0",
  "simple-git": "^3.20.0"
}
```

## Shared Packages

### packages/shared
Shared types, constants, validators used across all apps.

**Key Directories:**
```
packages/shared/
├── src/
│   ├── types/                      ← Shared TypeScript types
│   │   ├── company.ts
│   │   ├── agent.ts
│   │   ├── issue.ts
│   │   ├── heartbeat.ts
│   │   ├── actor.ts
│   │   ├── api.ts
│   │   └── index.ts
│   │
│   ├── constants/                  ← Configuration constants
│   │   ├── agent-roles.ts          ← CEO, engineer, designer, etc.
│   │   ├── agent-status.ts         ← Active, paused, idle, error, etc.
│   │   ├── issue-status.ts         ← Backlog, in-progress, done, etc.
│   │   ├── providers.ts            ← LLM providers (anthropic, openai, etc.)
│   │   ├── api-errors.ts           ← Error codes
│   │   └── index.ts
│   │
│   ├── validators/                 ← Zod validation schemas
│   │   ├── company.schema.ts
│   │   ├── agent.schema.ts
│   │   ├── issue.schema.ts
│   │   ├── api.schema.ts
│   │   └── index.ts
│   │
│   └── utils/                      ← Utility functions
│       ├── id-generator.ts         ← UUID generation
│       ├── date-utils.ts           ← Date formatting
│       ├── cost-calculator.ts      ← Cost computations
│       └── index.ts
│
├── package.json
└── tsconfig.json
```

**Dependencies:**
```json
{
  "zod": "^3.22.0"
}
```

### packages/adapters
Adapter registry and per-adapter command/output specifications.

**Key Directories:**
```
packages/adapters/
├── src/
│   ├── registry.ts                 ← Central adapter registry
│   │
│   ├── types/
│   │   ├── adapter.ts              ← IAdapter interface
│   │   ├── execution-request.ts
│   │   ├── session-codec.ts
│   │   └── index.ts
│   │
│   ├── claude/
│   │   ├── claude.adapter.ts
│   │   ├── claude.spec.ts          ← Command syntax, output parsing
│   │   └── index.ts
│   │
│   ├── codex/
│   ├── gemini/
│   ├── openai/
│   ├── openclaw-gateway/
│   ├── cursor/
│   ├── pi/
│   ├── process/
│   ├── http/
│   │
│   └── index.ts                    ← Export all adapters
│
├── package.json
└── tsconfig.json
```

**Contents:** Adapter command syntax, environment variable mappings, output parsing rules, session codec specifications.

### packages/adapter-utils
Shared utilities for all adapters.

**Key Directories:**
```
packages/adapter-utils/
├── src/
│   ├── session-codec/              ← Serialize/deserialize agent state
│   │   ├── encoder.ts
│   │   ├── decoder.ts
│   │   └── types.ts
│   │
│   ├── stream-parser/              ← Parse streaming responses
│   │   ├── json-parser.ts
│   │   ├── sse-parser.ts
│   │   └── error-parser.ts
│   │
│   └── utils/                      ← Common functions
│       ├── error-handling.ts
│       ├── retry-logic.ts
│       └── timeout-handler.ts
│
├── package.json
└── tsconfig.json
```

## Configuration Packages

### config/skills/
Agent instruction files (SKILL.md templates).

```
config/skills/
├── ceo/
│   └── SKILL.md                    ← CEO role instructions
│
├── engineer/
│   └── SKILL.md                    ← Engineer role instructions
│
├── designer/
│   └── SKILL.md                    ← Designer role instructions
│
├── marketer/
│   └── SKILL.md                    ← Marketer role instructions
│
└── ... (additional roles)
```

Each SKILL.md contains:
- Role description
- Responsibilities
- Available tools
- Goals and metrics
- Delegation guidelines
- Example workflows

### config/templates/
Company setup templates (initial config + agent roster).

```
config/templates/
├── saas-startup.json               ← SaaS company template
├── consulting.json                 ← Consulting firm template
├── ecommerce.json                  ← E-commerce store template
├── marketing-agency.json           ← Marketing agency template
└── ...
```

Example template structure:
```json
{
  "name": "SaaS Startup",
  "description": "Full software company with engineering and marketing",
  "agents": [
    {
      "role": "ceo",
      "name": "CEO Bot",
      "adapterType": "claude",
      "budgetMonthlyCents": 10000
    },
    {
      "role": "engineer",
      "name": "Lead Engineer",
      "adapterType": "claude",
      "reportsTo": "ceo"
    },
    ...
  ],
  "goals": [
    {
      "title": "Build MVP",
      "description": "...",
      "priority": 1
    }
  ]
}
```

## Test Package

### tests/e2e/
End-to-end tests using Playwright.

```
tests/e2e/
├── auth.spec.ts                    ← Signup, login, logout flows
├── company.spec.ts                 ← Create company, update settings
├── agents.spec.ts                  ← Hire agents, configure, monitor
├── tasks.spec.ts                   ← Create tasks, checkout, update
├── heartbeat.spec.ts               ← Manual & scheduled heartbeats
├── billing.spec.ts                 ← Cost tracking, budgets
└── fixtures/
    └── test-data.ts                ← Test companies, users, agents
```

## Dependencies & Build

### Root package.json
Defines workspaces and shared dev dependencies:

```json
{
  "name": "ai-orchestration-platform",
  "version": "0.1.0",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "devDependencies": {
    "turbo": "^1.10.0",
    "typescript": "^5.3.0",
    "vitest": "^1.1.0",
    "@types/node": "^20.10.0",
    "prettier": "^3.1.0",
    "eslint": "^8.55.0"
  },
  "scripts": {
    "dev": "turbo run dev --parallel",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "test:e2e": "turbo run test:e2e"
  }
}
```

### turbo.json
Turborepo build configuration:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
    },
    "dev": {
      "cache": false
    }
  }
}
```

## Dependency Graph

```
apps/backend
  ├── packages/shared (types, validators)
  ├── packages/adapters (adapter registry)
  └── packages/adapter-utils (streaming, codec)

apps/web
  ├── packages/shared (types)
  └── External: React, Tailwind, shadcn/ui

apps/executor
  ├── packages/adapters (adapter implementations)
  └── packages/adapter-utils (session codec, stream parsing)

packages/adapters
  └── packages/adapter-utils

packages/shared
  └── (no dependencies, root leaf)

packages/adapter-utils
  └── packages/shared (types only)

tests/e2e
  ├── apps/backend (API under test)
  └── apps/web (UI under test)
```

## Build Process

### Local Development
```bash
pnpm install                        # Install all dependencies
pnpm dev                            # Run all apps in dev mode
                                    # backend: localhost:3100
                                    # web: localhost:5173
                                    # executor: localhost:3200
```

### CI/CD (GitHub Actions)
```bash
1. pnpm install
2. pnpm typecheck                   # TypeScript compilation
3. pnpm lint                        # Linting
4. pnpm test                        # Unit + integration tests
5. pnpm build                       # Production builds
6. pnpm test:e2e                    # E2E tests (optional, slow)
7. Deploy to cloud (Railway/Fly.io)
```

### Docker Build (Production)
Multistage build:
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN pnpm install && pnpm build

# Stage 2: Production
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3100
CMD ["node", "dist/apps/backend/main.js"]
```

## Key Design Patterns

### CQRS (Command Query Responsibility Segregation)
- Separates read and write operations
- Commands for state mutations
- Queries for data reads
- Enables independent scaling of read/write paths

### Dependency Injection
- All dependencies injected via constructor
- NestJS @Injectable() decorator
- No global state or singletons
- Enables easy testing with mocks

### Clean Architecture
- Domain layer (pure TypeScript, no framework)
- Infrastructure layer (database, external services)
- Application layer (CQRS handlers, use cases)
- Presentation layer (HTTP controllers, DTOs)

### Repository Pattern
- ICompanyRepository interface (domain layer)
- CompanyRepository implementation (infrastructure layer)
- Enables testability and persistence abstraction

### Adapter Pattern
- IAdapter interface (packages/adapters)
- Per-LLM implementations (claude, openai, gemini, etc.)
- Adapter registry for dynamic selection

## File Naming Conventions

| File Type | Convention | Example |
|-----------|-----------|---------|
| TypeScript (backend) | kebab-case | `company.repository.ts` |
| React components | PascalCase | `CompanyCard.tsx` |
| Tests | `.spec.ts` suffix | `company.repository.spec.ts` |
| Database migrations | Timestamp prefix | `20240101120000_create_companies.ts` |
| Constants | UPPER_SNAKE_CASE | `MAX_AGENTS_PER_COMPANY.ts` |

## Asset Locations

- **Config files:** `config/` (skills/, templates/)
- **Environment files:** `.env.example` (never commit `.env`)
- **Docker files:** Root `Dockerfile`, per-app `Dockerfile` if needed
- **CI/CD workflows:** `.github/workflows/`
- **Documentation:** `docs/` (blueprints + generated)

## Code Statistics (Estimated)

| Package | Lines of Code | Files | Purpose |
|---------|---------------|-------|---------|
| backend/src | 15,000-20,000 | 150-200 | API server, CQRS, DB |
| web/src | 8,000-12,000 | 80-120 | React dashboard |
| executor/src | 3,000-5,000 | 30-50 | Agent executor |
| packages/shared | 2,000-3,000 | 30-50 | Types, validators |
| packages/adapters | 4,000-6,000 | 50-70 | Adapter implementations |
| tests | 5,000-8,000 | 50-80 | E2E tests |
| **TOTAL** | **37,000-54,000** | **380-570** | Complete platform |

## Technology Stack Summary

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Runtime** | Node.js | 20+ | Server runtime |
| **Framework** | NestJS | 10+ | Backend HTTP API |
| **Lang** | TypeScript | 5.3+ | Type-safe code |
| **Frontend** | React | 18+ | UI framework |
| **Build** | Vite | 5+ | Frontend bundler |
| **Database** | PostgreSQL | 15+ | Multi-tenant data |
| **ORM** | Drizzle | 0.30+ | Type-safe database |
| **Cache** | Redis | 7+ | Pub/sub events |
| **Auth** | Better Auth | Latest | Session management |
| **Validation** | Zod | 3.22+ | Runtime validation |
| **Testing** | Vitest | 1+ | Unit tests |
| **E2E** | Playwright | 1.40+ | Browser tests |
| **Execution** | Fly.io | - | Per-company VMs |
| **Storage** | AWS S3 | - | Logs, attachments |
| **Monitoring** | Pino | 8+ | Structured logging |

## Next Steps

1. Set up monorepo (Turborepo + pnpm)
2. Create database schema (Drizzle migrations)
3. Implement core CQRS handlers (create company, hire agent)
4. Build execution engine (HTTP to Fly.io VM)
5. Create adapter implementations (Claude, GPT, etc.)
6. Build React dashboard (company, agents, tasks)
7. Implement heartbeat scheduler
8. Add real-time events (WebSocket + Redis)
9. E2E testing and CI/CD
10. Deploy to production
