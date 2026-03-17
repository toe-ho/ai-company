# AI Company Platform

**Launch a fully autonomous AI-powered business for $2K/month instead of $250K/month.**

A web platform where non-technical entrepreneurs create and run AI companies staffed entirely by AI agents. Define a business goal, assemble a team (CEO, engineers, designers, marketers), and watch agents work autonomously 24/7.

## What This Is

- **Control Plane:** NestJS API + scheduler orchestrating agent execution, cost tracking, governance
- **Execution Plane:** Per-company Fly.io VMs running agent processes
- **Dashboard:** React web UI for team management, task assignment, real-time monitoring
- **Multi-tenant:** One database serves all users with row-level isolation

## Quick Facts

| | Details |
|---|---------|
| **Frontend** | React, Vite, Tailwind, shadcn/ui, React Query |
| **Backend** | Node.js 20+, NestJS, TypeScript, @nestjs/schedule |
| **Database** | PostgreSQL (Neon/Supabase), Drizzle ORM, 35+ tables |
| **Auth** | Better Auth (sessions), Agent JWT, encrypted API keys |
| **Execution** | Fly.io Machines per company VM |
| **Real-time** | Redis pub/sub + WebSocket |
| **Monorepo** | pnpm workspaces + Turborepo |
| **CI/CD** | GitHub Actions (typecheck → test → build → deploy) |

## Architecture

```
User (Browser)
    ↓ HTTPS / WebSocket (session cookie)
    ↓
NestJS Control Plane
  ├── Auth & Sessions (Better Auth)
  ├── Heartbeat Scheduler (@nestjs/schedule)
  ├── Execution Engine (dispatch to VMs via HTTP)
  ├── CQRS Bus (Commands & Queries)
  └── Real-time Events (Redis pub/sub)
    ↓
PostgreSQL (Neon)          Redis (Upstash)
Multi-tenant DB            Pub/sub only
35+ tables
    ↓
Fly.io Machines (Per-company VMs)
  ├── Agent Executor process
  ├── AI agent processes
  ├── Git workspace
  └── Persistent volumes
```

## Key Concepts

### Control Plane vs Execution Plane
- **Control Plane:** Scheduling, auth, cost tracking, governance, real-time events
- **Execution Plane:** Where agent code actually runs (per-company Fly.io VMs)

### Heartbeat Model
Agents run in discrete execution windows (heartbeats), not continuously. Prevents runaway costs, enables human oversight, simplifies recovery.

### CQRS Pattern
All writes are Commands, all reads are Queries. Clean separation of concerns.

### Multi-tenant by Design
Every entity has `companyId`. One database serves all users. Row-level query filtering. Users never see each other's data.

### Atomic Task Checkout
Only one agent can own a task at a time. `POST /issues/:id/checkout` returns 409 if already owned. Prevents double-work.

## Core Components

| Component | Purpose | Tech |
|-----------|---------|------|
| **API Server** | HTTP API, scheduling, orchestration | NestJS + TypeScript |
| **Web Dashboard** | User interface for company management | React + Vite |
| **Agent Executor** | Runs on Fly.io VM, executes agent processes | Node.js process manager |
| **Adapter Registry** | 9+ LLM adapters (Claude, GPT, Gemini, etc.) | Node.js modules |
| **Database** | Multi-tenant PostgreSQL | Drizzle ORM |

## Project Structure

```
apps/
├── backend/               ← NestJS API + Scheduler
├── web/                   ← React frontend (Vite)
└── executor/              ← Agent Executor (runs on Fly.io VM)
packages/
├── shared/                ← Types, constants, validators (Zod)
├── adapters/              ← 9+ agent runtime integrations
└── adapter-utils/         ← Shared adapter utilities
config/
├── skills/                ← Agent instruction files (SKILL.md)
└── templates/             ← Company templates
tests/                     ← E2E tests (Playwright)
```

## Getting Started

### Prerequisites
- Node.js 20+
- pnpm 9+
- PostgreSQL (local or Neon)
- Redis (local or Upstash)

### Setup (Planned)
```bash
# Install dependencies
pnpm install

# Build all packages
pnpm build

# Run API server (localhost:3100)
pnpm run dev --filter backend

# Run frontend (localhost:5173)
pnpm run dev --filter web
```

*Note: Source code does not exist yet. This is a blueprint-based documentation.*

## Documentation

All technical documentation is in `docs/blueprint/` organized by topic:

| Category | Files | Purpose |
|----------|-------|---------|
| **Product** | `01-product/` | What we're building, target users, business model |
| **AI System** | `02-ai-system/` | Execution engine, adapters, agent workflow |
| **Architecture** | `03-architecture/` | System design, CQRS, monorepo setup |
| **Data & API** | `04-data-and-api/` | Database schema, API endpoints |
| **Operations** | `05-operations/` | Auth, security, error handling, jobs |
| **Infrastructure** | `06-infrastructure/` | Deployment, CI/CD, testing |

**Quick Navigation:**
- [System Architecture Overview](docs/blueprint/03-architecture/09-system-architecture.md)
- [Backend Architecture & CQRS](docs/blueprint/03-architecture/11-backend-architecture.md)
- [Database Design](docs/blueprint/04-data-and-api/15-database-design.md)
- [AI Execution System](docs/blueprint/02-ai-system/02-ai-architecture.md)
- [Deployment Guide](docs/blueprint/06-infrastructure/24-deployment-and-infra.md)

## Key Design Decisions

1. **One VM per company** — Not per agent. Agents share workspace, costs stay low.
2. **API keys only** — Users bring own LLM keys. Never stored on VMs.
3. **Multi-tenant database** — One Postgres serves all users. Row-level isolation.
4. **Heartbeat scheduler** — Discrete execution windows prevent runaway costs.
5. **Atomic task checkout** — Only one agent owns a task (409 on conflict).
6. **Clean architecture** — Domain → Infrastructure → Application → Presentation.

## Deployment

**Control Plane:** Railway or Fly.io ($10-20/mo)
**Database:** Neon/Supabase ($0-25/mo)
**Redis:** Upstash ($0-10/mo)
**Storage:** AWS S3 ($1-5/mo)
**Per-Company VMs:** Fly.io Machines (billed to platform, marked up to users)

Total platform overhead: ~$11-62/mo

See [Deployment Guide](docs/blueprint/06-infrastructure/24-deployment-and-infra.md) for full setup.

## Business Model

- **Platform fee:** $49-99/month per user
- **Compute markup:** Fly.io Machines costs + 20-30% margin
- **LLM costs:** Users pay directly to Anthropic, OpenAI, Google
- **No recurring SaaS subscription** — One-time agent setup + monthly platform fee

## Development Workflow

### Phase 1: Core Platform (Weeks 1-4)
- Monorepo setup (Turborepo + pnpm)
- Database schema & migrations (Drizzle)
- Auth system (Better Auth)
- Basic CQRS structure (Commands, Queries, Handlers)

### Phase 2: AI Execution (Weeks 5-8)
- Adapter registry & implementations
- Agent executor on Fly.io VM
- Heartbeat scheduler
- Execution engine (HTTP → SSE)

### Phase 3: Frontend (Weeks 9-12)
- Dashboard (company overview, team management)
- Team builder (hire agents, configure)
- Task board & org chart

### Phase 4: Advanced Features (Weeks 13+)
- Company templates
- Approval workflows
- Cost analytics & budgets
- Self-hiring agents

See [Project Roadmap](docs/project-roadmap.md) for full phases.

## Code Standards

All code follows Clean Architecture principles:
- **Domain Layer:** Pure TypeScript, no framework imports
- **Infrastructure Layer:** Drizzle schemas, repository implementations
- **Application Layer:** Commands, Queries, event handlers
- **Presentation Layer:** HTTP controllers, DTOs, guards

Validation via Zod. Error handling via domain exceptions. CQRS for all state mutations.

See [Code Standards](docs/code-standards.md) for conventions.

## Testing

- **Unit Tests:** Vitest (handlers, services, repositories)
- **Integration Tests:** Drizzle + test database
- **E2E Tests:** Playwright (full user journeys)

Tests run on every commit via GitHub Actions. Must pass before merge.

## Production Checklist

- [ ] CI/CD pipeline fully working
- [ ] Database migrations tested
- [ ] API server deployed on Railway/Fly.io
- [ ] Fly.io account setup for per-company VMs
- [ ] Redis (Upstash) configured
- [ ] S3 bucket created
- [ ] SSL certificate configured
- [ ] Monitoring & alerting setup
- [ ] Load testing completed

## Support & Issues

See [System Architecture](docs/system-architecture.md) for component diagrams.
See [Code Standards](docs/code-standards.md) for architectural guidelines.
See [Deployment Guide](docs/deployment-guide.md) for infrastructure setup.

---

**Status:** Blueprint documentation phase. Source code to be implemented per development roadmap.
