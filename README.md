# AI Company Platform v2

A web platform enabling anyone to create and run a fully autonomous AI company. Users define a business goal; the platform assembles a team of AI agents that work 24/7 autonomously — writing code, designing UI, marketing, managing projects — without human intervention.

**Economics:** Build a 15-person team for ~$2K/month instead of $250K+/month. 140x cheaper. Same output. Always available.

---

## Quick Links

- **Project Overview & PDR:** [`docs/project-overview-pdr.md`](./docs/project-overview-pdr.md)
- **Codebase Summary:** [`docs/codebase-summary.md`](./docs/codebase-summary.md)
- **Code Standards:** [`docs/code-standards.md`](./docs/code-standards.md)
- **System Architecture:** [`docs/system-architecture.md`](./docs/system-architecture.md)
- **Project Roadmap:** [`docs/project-roadmap.md`](./docs/project-roadmap.md)
- **Deployment Guide:** [`docs/deployment-guide.md`](./docs/deployment-guide.md)
- **Design Guidelines:** [`docs/design-guidelines.md`](./docs/design-guidelines.md)

**Detailed specs:** [`docs/blueprint/`](./docs/blueprint/) (27 comprehensive spec documents)

---

## What This Platform Does

**For non-technical entrepreneurs:**
1. Create company (name, goal, budget)
2. Platform scaffolds AI team (CEO, engineers, designers, marketers)
3. Assign work (tasks/issues)
4. Agents execute autonomously, 24/7
5. Watch real-time dashboard as your company works

**Agent capabilities:**
- Write code (with execution environment)
- Design UI (with tool access)
- Create marketing campaigns
- Manage projects (coordinate with other agents)
- Hire additional agents (with approval)
- Track own costs and budget

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 19 + Vite + React Router 7 + React Query + Tailwind CSS 4 + shadcn/ui |
| **Backend** | NestJS + TypeScript + CQRS + Clean Architecture |
| **Database** | PostgreSQL (Neon managed) + Drizzle ORM |
| **Real-time** | WebSocket + Redis pub/sub (Upstash) |
| **Execution** | Fly.io VMs per company + Agent Executor (Express) |
| **Adapters** | Claude CLI, OpenClaw WebSocket, Process generic (V1: 3 of 9) |
| **Auth** | Better Auth (sessions) + JWT (agents) + encrypted API keys |
| **Testing** | Vitest + Playwright |
| **CI/CD** | GitHub Actions |
| **Monitoring** | Axiom (logs) + Sentry (errors) |

---

## Monorepo Structure

```
ai-orchestration-company-v2/
├── apps/
│   ├── backend/              NestJS API + Scheduler (CQRS, Clean Architecture)
│   ├── web/                  React 19 SPA (11 pages, Vite)
│   └── executor/             Express executor service (Fly.io VMs)
├── packages/
│   ├── shared/               Types, validators, enums (monorepo-wide)
│   ├── adapters/             Agent runtime integrations (Claude, OpenClaw, Process)
│   └── adapter-utils/        Shared adapter utilities (process spawn, WebSocket)
├── config/
│   ├── skills/               Agent instruction templates
│   └── templates/            Company/agent scaffolding templates
├── tests/                    E2E tests (Playwright)
├── docs/                     Operational documentation
├── docs/blueprint/           Detailed specs (27 documents, not modified)
├── plans/                    Implementation phases (19 phases, ~49h)
└── .github/workflows/        CI/CD pipelines
```

---

## Architecture Overview

### Control Plane (API Server)
**NestJS API** running on Railway or Fly.io (Port 3000)

Responsibilities:
- Company & agent management (CRUD)
- Task orchestration (checkout, assignment)
- Heartbeat scheduling (periodic execution trigger)
- Execution dispatch (to Fly.io VMs)
- Cost tracking & budgets
- Real-time WebSocket events
- Database (PostgreSQL, 35+ tables)

### Execution Plane (Fly.io VMs)
Per-company isolated VM running **Express executor** (Port 8080)

Responsibilities:
- Receive execution requests
- Load adapter (Claude CLI, OpenClaw, Process)
- Spawn agent process
- Stream results back to API (SSE)
- Track execution logs & costs
- Persistent workspace (git repos, files)

### Frontend (React SPA)
**Vite + React 19** deployed to Vercel or Fly.io

11 pages:
- Auth (login, signup, forgot password)
- Onboarding wizard (company setup, API keys, team)
- Dashboard (metrics, quick actions)
- Team management (agents, org chart)
- Agent detail (profile, history)
- Task board (kanban view)
- Task detail (comments, logs)
- Approvals (governance workflow)
- Cost dashboard (spending, forecasts)
- Settings (profile, API keys)

### Real-time Updates
WebSocket + Redis pub/sub for:
- Agent status changes
- Task progress updates
- Cost tracking
- Approval notifications

---

## Database Schema

**35+ tables** across 6 domains (all multi-tenant with `companyId`):

| Domain | Tables | Purpose |
|--------|--------|---------|
| **Org** | users, companies, invites | User accounts, company management |
| **Team** | agents, permissions, budgets, hierarchy | Agent definitions, roles, permissions |
| **Work** | issues, history, comments, attachments | Task management with full audit trail |
| **Execution** | heartbeat_schedules, runs, events | Scheduling & execution logs |
| **Costs** | cost_events, budget_alerts, monthly_billing | Token/compute cost tracking |
| **Infra** | api_keys, vms, migrations | Credentials, VM management, schema versions |

**Key features:**
- JSONB columns for flexible configs
- Full-text search on issues
- Drizzle ORM (type-safe migrations)
- Connection pooling via Neon

---

## Getting Started (Planned)

### Prerequisites
- Node.js 18+
- pnpm 8+
- PostgreSQL 14+ (local or Neon)
- Redis (local or Upstash)

### Local Setup

```bash
# Clone repo
git clone https://github.com/yourusername/ai-orchestration-company-v2.git
cd ai-orchestration-company-v2

# Install dependencies
pnpm install

# Copy environment files
cp .env.example .env.local
# Edit .env.local with your values

# Run database migrations
cd apps/backend
pnpm db:migrate
pnpm db:seed

# Start all services (backend, web, executor)
pnpm dev

# In separate terminals:
# Backend: http://localhost:3000
# Frontend: http://localhost:5173
# Executor: http://localhost:8080
```

### Development Commands

```bash
pnpm build          # Build all packages
pnpm test           # Run all tests
pnpm typecheck      # TypeScript check
pnpm lint           # ESLint check
pnpm dev            # Start all dev servers
pnpm db:migrate     # Run database migrations
pnpm db:seed        # Seed test data
```

---

## Implementation Status

**Project stage:** Greenfield (no code yet)
**Phase:** All 19 implementation phases pending
**Effort estimate:** 49 hours total

### Phase Overview (See `docs/project-roadmap.md` for details)

| Phase | Name | Hours | Status |
|-------|------|-------|--------|
| 1-3 | Foundation (monorepo, shared, db) | 6h | pending |
| 4-7 | Backend core (domain, repos, auth, services) | 10h | pending |
| 8-11 | CQRS & modules | 11h | pending |
| 12-13 | Controllers, real-time | 6h | pending |
| 14-15 | Executor, adapters | 5h | pending |
| 16-17 | Frontend (foundation, 11 pages) | 8h | pending |
| 18-19 | Testing, CI/CD, deployment | 5h | pending |

**Ready to begin:** Phase 1 (Monorepo Foundation)

---

## Key Design Decisions

1. **V1 Adapters:** Claude CLI + OpenClaw WebSocket + Process generic (defer 6 others)
2. **Tooling:** React Router 7 + Vitest (not Jest)
3. **Clean Architecture:** 4 strict layers (Domain, Infrastructure, Application, Presentation)
4. **CQRS:** Separate commands & queries for async operations
5. **Multi-tenant:** Every query filters by `companyId`, per-company VMs
6. **Heartbeat model:** Discrete execution windows (not continuous agents)
7. **DB driver:** node-postgres (pg), not Neon serverless
8. **Dev mode:** Local executor at localhost:8080 (no Fly.io needed)
9. **Partitioning:** Defer heartbeat_run_events partitioning post-MVP
10. **Clean-room:** Fresh v2 (no legacy v1 code)

---

## Security & Compliance

- **Multi-tenant isolation:** Database row-level security + isolated VMs
- **API key vault:** AES-256-GCM encryption at rest
- **Auth:** Better Auth (sessions) + JWT (agents) + API keys
- **Audit logs:** All user actions logged with actor, action, timestamp
- **Cost controls:** Budget limits per agent, automatic pause on exceed
- **Approvals:** Governance workflows for high-risk decisions (hiring, budget changes)

---

## Scaling Path

### MVP (Current)
- 1 API instance
- 1 DB (Neon)
- 1 Redis (Upstash)
- 1 VM per company (auto-hibernates)
- ~100 RPS capacity

### Growth Phase
- 2-3 API instances (load balanced)
- DB read replicas
- Redis cluster
- Multi-region VMs
- ~1000 RPS capacity

### Enterprise
- 10+ API instances (auto-scaling)
- Dedicated PostgreSQL cluster
- Distributed caching
- Multi-region, multi-cloud
- 10K+ RPS capacity

---

## Documentation

### Operational Docs (in `docs/`)
- **`project-overview-pdr.md`** — Product vision, user journeys, domain concepts, success metrics
- **`codebase-summary.md`** — Monorepo structure, package descriptions, file organization
- **`code-standards.md`** — TypeScript conventions, naming, architecture patterns, testing
- **`system-architecture.md`** — Control plane, execution plane, communication patterns, security
- **`project-roadmap.md`** — 19 implementation phases, dependencies, milestones
- **`deployment-guide.md`** — Cloud setup, environment variables, CI/CD, monitoring
- **`design-guidelines.md`** — UI/UX principles, component library, page layouts

### Detailed Specs (in `docs/blueprint/`, not modified during implementation)
- 27 comprehensive spec documents covering product, architecture, API, infrastructure, etc.

---

## Contributing

See `docs/code-standards.md` for:
- TypeScript conventions (strict mode, naming, imports)
- Clean Architecture layer rules
- CQRS patterns
- Testing standards (Vitest, Playwright)
- Git commit format (conventional commits)
- Code review checklist

**Pre-push checklist:**
- [ ] TypeScript compiles (`tsc --noEmit`)
- [ ] Tests pass (`pnpm test`)
- [ ] Linting passes (`pnpm lint`)
- [ ] No secrets in code
- [ ] Multi-tenant filters on DB queries
- [ ] File size < 200 LOC

---

## Monitoring & Observability

- **Logs:** Axiom (structured logging)
- **Errors:** Sentry (error tracking & source maps)
- **Metrics:** Datadog or Prometheus
- **Uptime:** Railway/Fly.io health checks

---

## Support

- **Documentation:** See `/docs` directory
- **Issues:** GitHub Issues
- **Spec Questions:** Refer to `docs/blueprint/` for detailed design decisions

---

## License

Proprietary. Internal use only.

---

## Project Info

- **Repository:** `ai-orchestration-company-v2`
- **Language:** TypeScript 100%
- **Node version:** 18+
- **Package manager:** pnpm
- **Monorepo:** pnpm workspaces + Turborepo
- **Created:** 2026-03-17
- **Status:** Greenfield (ready to begin Phase 1)

---

For detailed implementation guidance, see the full documentation in `docs/` and implementation plan in `plans/260317-1118-blueprint-implementation/`.

**Next steps:** Start Phase 1 (Monorepo Foundation). Estimated 2 hours to complete.
