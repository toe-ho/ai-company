# Project Overview & Product Development Requirements (PDR)

## Product Vision

A web platform enabling **anyone to create and run a fully autonomous AI company**. Users define a business goal; the platform assembles a team of AI agents (CEO, engineers, designers, marketers) that work 24/7 autonomously — writing code, designing UI, marketing, managing projects — without human intervention.

**Economics:** 15-person startup: $250K+/month (human). AI Company Platform: ~$2K/month (agents). **140x cheaper. Same output. 24/7 availability.**

---

## Target Users & Problems Solved

### Primary Users
**Non-technical entrepreneurs** with business ideas but insufficient capital for human teams.

### Problems Solved
| Problem | Before | With Platform |
|---------|--------|---------------|
| Team assembly cost | $250K+/month | $2K/month |
| Time-to-launch | Weeks/months | Days |
| 24/7 availability | Requires night shift | Always working |
| Scaling | Hire/train/pay | Auto-scale agents |
| Team management overhead | Continuous | Minimal dashboard |

### What It's NOT
- **Not a chatbot.** Agents have persistent roles, hierarchies, budgets.
- **Not a code generator.** Platform orchestrates teams of specialized AI agents.
- **Not self-hosted.** Cloud-only; agent execution on Fly.io VMs.
- **Not LLM agnostic.** Users bring their own API keys (Claude, GPT-4, Gemini, etc.).

---

## Domain Concepts

### Company
Organizational unit representing an AI-powered business. Owns:
- Goals (mission, OKRs)
- Budget allocation & spending limits
- Agent team (org chart with roles, responsibilities, permissions)
- Issues/tasks (unit of work)
- API keys (for external LLM APIs)
- Execution environment (Fly.io VM per company)

### Agent
AI worker with:
- Role (CEO, CTO, Engineer, Designer, Marketer, etc.)
- Adapter type (Claude CLI, OpenClaw WebSocket, Process generic)
- Budget limit (token + compute allowance)
- Capabilities/permissions (code access, approval authority, hiring rights)
- Reporting chain (manager, direct reports)
- Execution state (idle, running, blocked, error)

### Heartbeat
Discrete execution window (not continuous). Agent wakes, processes tasks, reports, exits. Triggered by:
- Scheduled interval (e.g., every 5 minutes)
- Manual trigger (user request)
- Event subscription (new task, manager message, etc.)

Key: Heartbeats enable atomic checkouts, cost tracking, and isolation.

### Issue/Task
Unit of work with status flow: **backlog** → **todo** → **in_progress** → **done**. Properties:
- Title, description, acceptance criteria
- Assigned agent + reporter
- Priority, due date
- Estimated effort (tokens, time)
- Comments/activity log

Atomic checkout: Only one agent can work at a time; prevents duplicate work.

### Adapter
Pluggable integration to specific AI runtime:
- **Claude CLI:** spawn process, send stdin/stdout
- **OpenClaw WebSocket:** bidirectional streaming
- **Process (generic):** generic subprocess executor

V1 ships 3 adapters; extensible to 9+ (OpenAI, Gemini, Cursor, Pi, etc.).

### Execution Engine
Service that:
- Triggers heartbeats via scheduler
- Dispatches work requests to Fly.io VMs
- Streams SSE (Server-Sent Events) results back to API
- Handles adapter routing, timeouts, retries
- Logs execution traces for debugging

### Multi-tenant Isolation
**Every database table includes `companyId`**. No data leakage across orgs. Fly.io VMs are per-company (auto-hibernating).

---

## Key User Journeys

### 1. Onboarding & Company Creation
1. User signs up (email + password via Better Auth)
2. Creates company (name, business goal)
3. Configures budget & API keys (OpenAI, Anthropic, etc.)
4. Platform scaffolds initial agent team from templates
5. Dashboard ready; agents idle, awaiting first task

### 2. Running Tasks
1. User creates issue (UI form or API)
2. Issue enters backlog
3. User assigns to agent + sets priority
4. Next heartbeat: agent wakes, checks out task
5. Agent executes (via adapter), makes tool calls, writes results
6. User sees real-time progress (WebSocket updates)
7. Agent completes; task moves to done

### 3. Team Management
1. User views org chart (CEO → CTO → Engineers)
2. Configures agent budgets, permissions
3. Enables agent-hiring workflow (agents propose hires; user approves)
4. Monitors agent performance metrics (tasks completed, cost/task, error rate)

### 4. Cost Control
1. Dashboard shows real-time spending (tokens, compute)
2. Alerts when agent nears budget
3. User can pause, terminate, or increase budget
4. Monthly bill estimates

### 5. Approvals & Governance
1. Agents can propose hires, budget adjustments, strategy changes
2. Approval queue (user reviews + accepts/rejects)
3. Escalation paths (CEO → user for high-value decisions)
4. Audit log of all decisions

---

## Technical Architecture

### Control Plane (API Server)
**Stack:** NestJS + TypeScript + PostgreSQL (Neon)

**Layers (Clean Architecture):**
1. **Presentation:** Controllers, DTOs, Guards, Interceptors
2. **Application:** Commands, Queries, Event Handlers, Services
3. **Infrastructure:** Drizzle Schemas, Repositories, External Clients
4. **Domain:** Entities, Interfaces, Enums (zero framework imports)

**Modules:**
- SharedModule (auth, logging, config)
- ApiModule (companies, agents, issues, costs)
- SchedulerModule (heartbeat scheduling, execution dispatch)
- RealtimeModule (WebSocket, pub/sub)

**Key Patterns:**
- CQRS (Command Query Responsibility Segregation) for async operations
- Domain-Driven Design (bounded contexts per feature)
- Repository pattern for data access
- Zod validators for DTOs

### Execution Plane (Fly.io VMs)
**Stack:** Express + Node.js

**Per-company VM:**
- Executor service (receives dispatch requests, spawns adapter processes)
- Workspace (persistent git repos, files)
- Adapter registry (Claude, OpenClaw, Process)
- Memory/cost tracking per run

**Scaling:** Auto-hibernates when idle; spins up on next heartbeat.

### Frontend (Web Dashboard)
**Stack:** React 19 + Vite + Tailwind CSS 4 + shadcn/ui + React Router 7 + React Query

**11 Pages:**
- Auth (Login, Signup, Forgot Password)
- Onboarding Wizard (Company setup, API keys, initial team)
- Dashboard (Metrics, quick actions, recent activity)
- Team Management (Agents, org chart, configure roles/budgets)
- Agent Detail (Profile, capabilities, execution history)
- Task Board (Kanban view, filter, assign)
- Task Detail (Comments, execution logs, approval workflow)
- Approvals (Pending decisions, voting)
- Cost Dashboard (Spending trends, budget allocation, forecasts)
- Settings (Profile, API key management, notifications)

**Real-time:** WebSocket for live agent status, task progress, cost updates.

### Database (PostgreSQL)
**35+ tables** across 6 domains:

| Domain | Tables | Purpose |
|--------|--------|---------|
| **Org** | users, companies, invites, roles | User accounts, company management |
| **Team** | agents, agent_permissions, costLimits | Agent definitions, permissions |
| **Work** | issues, issueHistory, comments, assignments | Task management |
| **Execution** | heartbeatSchedules, heartbeatRuns, heartbeatRunEvents | Execution logs |
| **Costs** | costEvents, budgetAlerts | Token/compute tracking |
| **Infra** | companyApiKeys, companyVms, migrations | Credentials, VMs, schema versions |

**Key Pattern:** Multi-tenant (every table includes `companyId`). JSONB fields for flexible configs (adapterConfig, runtimeConfig, usageJson).

---

## Design Principles

### 1. Non-Tech-First UX
- Entrepreneurs with no technical background should understand the product.
- Avoid jargon; use business concepts (CEO, tasks, budgets).
- Guided onboarding for non-technical users.

### 2. Progressive Disclosure
- Beginner view: Simple dashboard with essential actions.
- Advanced view: Deep execution logs, adapter configs, fine-grained permissions.
- Settings only shown when needed.

### 3. Real-time Transparency
- Users watch agents work (live execution logs, WebSocket updates).
- Builds trust; shows ROI.

### 4. Multi-tenant Isolation
- No data bleeding between companies.
- Each company gets isolated Fly.io VM.
- Cryptographic API key storage.

### 5. Fail-Safe Governance
- Users retain override authority.
- Approval workflows for high-risk agent decisions (hiring, large budget requests).
- Cost kill-switch; agents pause if budget exceeded.
- Audit logs for compliance.

### 6. Cost Transparency
- Real-time token + compute cost tracking.
- Per-agent budgets with alerts.
- Monthly forecasts.
- Usage breakdown by task, agent, tool.

---

## Success Metrics (KPIs)

### User Adoption
- **Sign-ups:** 100 in 6 months, 500 in 12 months
- **Active companies:** >80% monthly usage
- **Avg team size:** 5-15 agents per company

### Business Economics
- **ARR:** $50K (25 paying companies @ $2K/month)
- **CAC payback:** <12 months
- **Churn:** <5% monthly

### Platform Reliability
- **API uptime:** 99.9%
- **Heartbeat success rate:** >95%
- **Agent error recovery:** Auto-retry + user alerts
- **DB query latency (p95):** <100ms

### User Experience
- **Onboarding time:** <15 minutes to first agent heartbeat
- **Task completion latency:** <5 minutes (agent wakeup + execution)
- **Dashboard load time:** <2 seconds
- **User satisfaction (NPS):** >50

### Security
- **Data breach incidents:** 0
- **Unauthorized access attempts:** Logged, blocked
- **Audit log coverage:** 100% of user actions
- **API key rotation:** Monthly automation

---

## Implementation Phases

**Total effort:** 49 hours across 19 phases.

| Phase | Name | Hours | Status |
|-------|------|-------|--------|
| 1-3 | Foundation (monorepo, shared, db) | 6h | pending |
| 4-7 | Backend core (domain, repos, auth, services) | 10h | pending |
| 8-11 | CQRS & modules | 11h | pending |
| 12-13 | Controllers, real-time | 6h | pending |
| 14-15 | Executor, adapters | 5h | pending |
| 16-17 | Frontend (foundation, 11 pages) | 8h | pending |
| 18-19 | Testing, CI/CD, deployment | 5h | pending |

**See:** `/plans/260317-1118-blueprint-implementation/plan.md` for phase breakdown, dependencies, and detailed requirements.

---

## Deployment & Infrastructure

### Cloud Providers
- **API Server:** Railway or Fly.io
- **Database:** Neon (PostgreSQL managed)
- **File Storage:** S3
- **Redis:** Upstash (pub/sub)
- **Agent VMs:** Fly.io Machines (auto-hibernating)

### Environment & Config
- GitHub Actions CI/CD (typecheck → unit tests → build → E2E)
- Environment variables for secrets (API keys, DB credentials, etc.)
- Docker containerization for reproducible builds

### Monitoring
- Application logs → Axiom or equivalent
- Database slow query logs
- Error tracking (Sentry or equivalent)
- Billing/cost alerts

---

## Constraints & Decisions

1. **Clean-room v2:** No legacy code; starts fresh from blueprint specs only.
2. **V1 Adapters:** Claude + Process + OpenClaw only (defer 6 others to v1.1).
3. **Tooling:** React Router 7 + Vitest (not Jest).
4. **DB Driver:** node-postgres (pg), not Neon serverless.
5. **Dev Mode:** Local executor at localhost:8080 (no mock execution).
6. **Partitioning:** Defer heartbeat_run_events partitioning to post-MVP.

---

## Next Steps

1. **Validate blueprint specs** against team availability/timeline.
2. **Spin up monorepo** (Phase 1): pnpm workspaces, Turborepo.
3. **Begin database design** (Phase 3): 35 tables, migrations.
4. **Parallel backend + frontend tracks** (Phases 4-17).
5. **Integration & testing** (Phases 18-19).

**Project status:** All 19 phases pending. Ready to begin Phase 1.

---

**Version:** 1.0
**Last Updated:** 2026-03-17
**Maintainer:** Development team
**See Also:** `./docs/blueprint/` for detailed specs (27 docs), `./plans/260317-1118-blueprint-implementation/` for phase guides.
