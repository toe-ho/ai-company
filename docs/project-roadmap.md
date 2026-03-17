# Project Roadmap & Implementation Phases

AI Company Platform v2 implementation: 19 phases, ~49 hours total effort, all pending.

---

## Phase Overview

| # | Phase | Description | Hours | Status | Progress |
|---|-------|-------------|-------|--------|----------|
| 1 | Monorepo Foundation | pnpm workspaces, Turborepo, package structure | 2h | pending | 0% |
| 2 | Shared Package | Types, validators, constants, enums | 1h | pending | 0% |
| 3 | Database Schema & Migrations | 35 tables, Drizzle ORM setup | 3h | pending | 0% |
| 4 | Domain Layer | Entities, interfaces, enums, exceptions | 2h | pending | 0% |
| 5 | Infrastructure - Repositories | Repository implementations, Drizzle queries | 3h | pending | 0% |
| 6 | Auth Layer | JWT, API keys, Better Auth integration | 3h | pending | 0% |
| 7 | Application - Core Services | ExecutionEngine, CostCalculator, etc. | 2h | pending | 0% |
| 8 | CQRS Handlers (Core) | Commands/Queries for companies, agents, issues | 4h | pending | 0% |
| 9 | CQRS Handlers (Supporting) | Commands/Queries for costs, approvals, activity | 3h | pending | 0% |
| 10 | Domain Events | Event definitions, handlers, pub/sub | 2h | pending | 0% |
| 11 | Cross-cutting & Modules | Shared module, interceptors, filters, guards | 2h | pending | 0% |
| 12 | Controllers & DTOs | REST endpoints, input/output contracts | 4h | pending | 0% |
| 13 | Real-time & Scheduler | WebSocket, Redis pub/sub, heartbeat scheduler | 2h | pending | 0% |
| 14 | Executor App | Express executor service, adapter registry | 3h | pending | 0% |
| 15 | Adapters (V1: 3 of 9) | Claude CLI, OpenClaw WebSocket, Process generic | 2h | pending | 0% |
| 16 | Frontend Foundation | Vite, React Router, React Query, Tailwind setup | 3h | pending | 0% |
| 17 | Frontend Pages | 11 pages (auth, dashboard, team, tasks, etc.) | 5h | pending | 0% |
| 18 | Testing | Unit tests (Vitest), E2E tests (Playwright) | 3h | pending | 0% |
| 19 | CI/CD & Deployment | GitHub Actions, Docker, deployment guides | 2h | pending | 0% |
| | **TOTAL** | | **49h** | **0%** | **0%** |

---

## Dependency Graph

```
Foundation
├─ Phase 1: Monorepo Foundation (2h)
│  └─ Phase 2: Shared Package (1h)
│     ├─ Phase 3: Database Schema (3h) ────────────────┬──────────────────┐
│     │  ├─ Phase 4: Domain Layer (2h) ────────────────┤                  │
│     │  │  └─ Phase 5: Infrastructure (3h) ──────────┬┤                  │
│     │  │                                             ││                  │
│     │  └─ Phase 6: Auth Layer (3h) ─────────────────┤├──────────────────┤
│     │                                                ││                  │
│     └─ Phase 7: Core Services (2h) ────────────────┬┤├──────────────────┤
│        ├─ Phase 8: CQRS Handlers (4h) ────────────┐││├──────────────────┤
│        │  └─ Phase 9: Supporting Handlers (3h) ──┐│││├──────────────────┤
│        │                                          │││││                  │
│        └─ Phase 10: Domain Events (2h) ───────┬──┘│││├──────────────────┤
│           ├─ Phase 11: Cross-cutting (2h) ────┘───┘││├──────────────────┤
│           │                                        ││                  │
│           └─ Phase 12: Controllers (4h) ──────────┬┘├──────────────────┤
│                                                    │├─ Phase 13: Real-time (2h)
│  Phase 14: Executor App (3h) ────────────────────┼┤
│  └─ Phase 15: Adapters (2h) ─────────────────────┘│
│                                                    │
│  Phase 16: Frontend Foundation (3h) ──────────────┤
│  └─ Phase 17: Frontend Pages (5h) ────────────────┤
│                                                    │
│  Phase 18: Testing (3h) ───────────────────────────┤
│  Phase 19: CI/CD & Deployment (2h) ────────────────┘
```

**Critical Path:** 1 → 2 → 3 → 4 → 5 → 7 → 8 → 10 → 11 → 12 → 13 (26 hours)
**Frontend Path:** 2 → 16 → 17 → 18 (13 hours)
**Executor Path:** 2 → 14 → 15 (7 hours)

---

## Detailed Phase Breakdown

### Phase 1: Monorepo Foundation (2h)
**Goal:** Set up pnpm workspaces, Turborepo, root package structure.

**Deliverables:**
- `pnpm-workspace.yaml` (workspace definition)
- `turbo.json` (Turborepo config: task caching, pipeline)
- Root `package.json` (scripts: build, test, dev, lint)
- `.npmrc` (pnpm config)
- TypeScript root `tsconfig.json`
- ESLint/Prettier config

**Status:** pending

### Phase 2: Shared Package (1h)
**Goal:** Create @ai-company/shared with types, validators, enums, constants.

**Deliverables:**
- `packages/shared/src/types/` (Company, Agent, Issue, Execution, Cost types)
- `packages/shared/src/schemas/` (Zod validators)
- `packages/shared/src/enums/` (Role, Status, Permission, etc.)
- `packages/shared/src/constants/` (defaults, config values)
- Barrel exports (`index.ts`)

**Status:** pending

### Phase 3: Database Schema & Migrations (3h)
**Goal:** Define all 35+ tables, Drizzle schemas, migrations.

**Deliverables:**
- `apps/backend/src/infrastructure/database/schema.ts` (All table definitions)
- `apps/backend/src/infrastructure/database/migrations/` (0001_init.ts, 0002_add_agents.ts, etc.)
- Migration runner setup
- Seed data script (test companies, agents, issues)

**Tables:**
- Core: users, companies, company_api_keys, company_vms
- Team: agents, agent_permissions, agent_cost_limits, agent_hierarchy
- Work: issues, issue_history, issue_comments, issue_attachments
- Execution: heartbeat_schedules, heartbeat_runs, heartbeat_run_events
- Costs: cost_events, budget_alerts, monthly_billing
- Activity: activity_log, approvals, audit_events

**Status:** pending

### Phase 4: Domain Layer (2h)
**Goal:** Entities, repository interfaces, enums, exceptions (no framework imports).

**Deliverables:**
- `src/domain/entities/` (Company, Agent, Issue, Execution entities)
- `src/domain/repositories/` (ICompanyRepository, IAgentRepository, etc.)
- `src/domain/enums/` (AgentRole, IssueStatus, ExecutionState, etc.)
- `src/domain/exceptions/` (CompanyNotFoundException, BudgetExceededException, etc.)
- `src/domain/types.ts` (domain-level type definitions)

**Status:** pending

### Phase 5: Infrastructure - Repositories (3h)
**Goal:** Implement repositories using Drizzle ORM, handle queries, mutations.

**Deliverables:**
- `src/infrastructure/repositories/` (CompanyRepository, AgentRepository, IssueRepository, etc.)
- Query implementations (findById, findMany, findByCompanyId, etc.)
- Mutation implementations (save, update, delete with multi-tenant filters)
- Index usage optimization

**Status:** pending

### Phase 6: Auth Layer (3h)
**Goal:** User sessions (Better Auth), JWT for agents, API key management.

**Deliverables:**
- Better Auth integration (email/password, OAuth)
- JWT generation/validation for agent authentication
- API key encryption/decryption (AES-256-GCM)
- Guards: JwtAuthGuard, ApiKeyAuthGuard
- Middleware: extract user/agent from request

**Status:** pending

### Phase 7: Application - Core Services (2h)
**Goal:** Business logic services (ExecutionEngine, CostCalculator, etc.).

**Deliverables:**
- `src/application/services/execution-engine.service.ts` (Dispatch to Fly.io VMs)
- `src/application/services/cost-calculator.service.ts` (Token → USD conversion)
- `src/application/services/workspace-manager.service.ts` (Git repos, files)
- `src/application/services/adapter-registry.service.ts` (Adapter loading)

**Status:** pending

### Phase 8: CQRS Handlers (Core) (4h)
**Goal:** Commands & Queries for companies, agents, issues.

**Deliverables:**
- Commands:
  - CreateCompanyCommand + handler
  - CreateAgentCommand + handler
  - CreateIssueCommand + handler
  - UpdateIssueStatusCommand + handler
  - CheckoutIssueCommand + handler
  - ExecuteHeartbeatCommand + handler

- Queries:
  - GetCompanyQuery + handler
  - GetAgentsQuery + handler
  - GetIssuesQuery + handler
  - GetCostBreakdownQuery + handler

**Status:** pending

### Phase 9: CQRS Handlers (Supporting) (3h)
**Goal:** Commands & Queries for costs, approvals, activity.

**Deliverables:**
- Commands:
  - RecordCostEventCommand + handler
  - CreateApprovalCommand + handler
  - ApproveDecisionCommand + handler
  - RejectDecisionCommand + handler
  - LogActivityCommand + handler

- Queries:
  - GetBudgetAlerts + handler
  - GetMonthlyBillingQuery + handler
  - GetApprovalQueueQuery + handler
  - GetAuditLogQuery + handler

**Status:** pending

### Phase 10: Domain Events (2h)
**Goal:** Define domain events, event handlers, pub/sub integration.

**Deliverables:**
- Event definitions (CompanyCreatedEvent, AgentExecutedEvent, IssueCompletedEvent, etc.)
- Event handlers (publish to Redis, store in activity_log)
- EventBus integration (NestJS event bus)
- Pub/sub subscribers (Redis integration)

**Status:** pending

### Phase 11: Cross-cutting & Modules (2h)
**Goal:** Shared module, interceptors, filters, guards, utilities.

**Deliverables:**
- SharedModule (global providers)
- GlobalExceptionFilter (error handling)
- LoggingInterceptor (request/response logging)
- CacheInterceptor (response caching)
- JwtAuthGuard, ApiKeyAuthGuard
- Utilities (generateId, formatDate, etc.)

**Status:** pending

### Phase 12: Controllers & DTOs (4h)
**Goal:** REST endpoints and request/response contracts.

**Deliverables:**
- Controllers:
  - CompanyController (create, read, update, delete, list)
  - AgentController (create, read, update, delete, list, config)
  - IssueController (create, read, update, list, comment)
  - CostController (get breakdown, usage, billing)
  - ApprovalController (list, approve, reject)

- DTOs:
  - CreateCompanyDTO, UpdateCompanyDTO, CompanyResponseDTO
  - CreateAgentDTO, UpdateAgentDTO, AgentResponseDTO
  - CreateIssueDTO, UpdateIssueDTO, IssueResponseDTO
  - CostEventDTO, BudgetAlertDTO, etc.

**Status:** pending

### Phase 13: Real-time & Scheduler (2h)
**Goal:** WebSocket setup, Redis pub/sub, heartbeat scheduler.

**Deliverables:**
- WebSocket gateway (Socket.io or raw WS)
- Pub/sub integration (Redis channels)
- Heartbeat scheduler (@nestjs/schedule)
- Event streaming (SSE from executor)
- Real-time event types (agent_status, issue_updated, cost_update, etc.)

**Status:** pending

### Phase 14: Executor App (3h)
**Goal:** Express service for Fly.io VMs, adapter registry, cost tracking.

**Deliverables:**
- Express app setup (port 8080)
- Routes: /execute, /status, /logs, /workspace
- Adapter registry (load adapters dynamically)
- Execution request handler
- Cost tracking + token counting
- Error handling & retry logic

**Status:** pending

### Phase 15: Adapters (V1: 3 of 9) (2h)
**Goal:** Implement Claude CLI, OpenClaw WebSocket, Process generic adapters.

**Deliverables:**
- `packages/adapters/src/claude-cli/` (spawn, stdout capture, token count)
- `packages/adapters/src/openclaw-websocket/` (WS connect, message handling)
- `packages/adapters/src/process-generic/` (subprocess, stdin/stdout)
- Adapter interface implementation
- Error handling (TimeoutError, AdapterError)
- Cost calculators per adapter

**Status:** pending

### Phase 16: Frontend Foundation (3h)
**Goal:** Vite, React 19, React Router, React Query, Tailwind CSS, shadcn/ui setup.

**Deliverables:**
- Vite config (React plugin, HMR)
- React Router 7 setup (layout, loaders, actions)
- React Query configuration (cache time, retry policy)
- Tailwind CSS 4 config
- shadcn/ui component library
- API client setup (axios + interceptors)
- Auth context (session management)

**Status:** pending

### Phase 17: Frontend Pages (5h)
**Goal:** Implement 11 pages (auth, dashboard, team, tasks, etc.).

**Deliverables:**
- Auth pages (Login, Signup, Forgot Password, Verify Email)
- Onboarding wizard (Company setup, API keys, initial team)
- Dashboard (Metrics, quick actions, recent activity)
- Team Management (Agent list, org chart, configure)
- Agent Detail (Profile, capabilities, execution history)
- Task Board (Kanban, filter, assign)
- Task Detail (Comments, execution logs, approval)
- Approvals (Pending decisions, voting)
- Cost Dashboard (Spending trends, forecasts)
- Settings (Profile, API keys, notifications)

**Status:** pending

### Phase 18: Testing (3h)
**Goal:** Unit tests (Vitest), integration tests, E2E tests (Playwright).

**Deliverables:**
- Unit tests for services, repositories, utilities (>80% coverage)
- Integration tests (DB + API layer)
- E2E tests: user journeys (signup, create company, assign task, execute)
- Test data generators (factories)
- Mock adapters for testing

**Status:** pending

### Phase 19: CI/CD & Deployment (2h)
**Goal:** GitHub Actions pipelines, Docker, deployment guides.

**Deliverables:**
- GitHub Actions workflows:
  - Typecheck (`tsc --noEmit`)
  - Lint (ESLint)
  - Unit tests (`vitest run`)
  - Build (`pnpm build`)
  - E2E tests (`playwright test`)
  - Deploy to prod

- Docker setup:
  - Backend Dockerfile (NestJS)
  - Executor Dockerfile (Express)
  - Frontend (Vercel or static hosting)

- Deployment guides:
  - Environment variables
  - Database migrations
  - Rollback procedures
  - Monitoring setup

**Status:** pending

---

## Milestones & Key Dates

### M1: Backend Foundation (Phases 1-7) — ~10 hours
**Completion Target:** Week 1
**Criteria:**
- Monorepo fully set up
- Database schema finalized + migrations working
- Domain layer + repositories operational
- Auth system tested

### M2: Backend API (Phases 8-13) — ~13 hours
**Completion Target:** Week 2
**Criteria:**
- All CQRS handlers implemented
- REST API endpoints functional
- Real-time WebSocket working
- Heartbeat scheduler firing

### M3: Executor & Adapters (Phases 14-15) — ~5 hours
**Completion Target:** Week 2-3
**Criteria:**
- Executor service running on Fly.io
- 3 adapters working (Claude, OpenClaw, Process)
- Cost tracking operational

### M4: Frontend (Phases 16-17) — ~8 hours
**Completion Target:** Week 3
**Criteria:**
- All 11 pages implemented
- API integration working
- WebSocket real-time updates

### M5: Testing & Deployment (Phases 18-19) — ~5 hours
**Completion Target:** Week 4
**Criteria:**
- >80% code coverage
- E2E tests passing
- CI/CD pipelines green
- Ready for production

---

## Success Criteria

### Functional Completeness
- [ ] Users can sign up and create companies
- [ ] Agents can be created and configured
- [ ] Issues can be assigned and executed
- [ ] Real-time dashboard updates
- [ ] Cost tracking accurate
- [ ] Approval workflows working
- [ ] All 11 frontend pages functional

### Code Quality
- [ ] No TypeScript errors (strict mode)
- [ ] >80% unit test coverage
- [ ] All E2E journeys pass
- [ ] No console errors in browser
- [ ] Code follows standards (see code-standards.md)

### Performance
- [ ] API response time (p95): <200ms
- [ ] Frontend load time: <3s
- [ ] Heartbeat execution: <10s
- [ ] Database queries (p95): <100ms

### Security
- [ ] No data leaks across companies
- [ ] API keys encrypted + never logged
- [ ] JWT validation on all agent requests
- [ ] Audit logs complete

### Deployment
- [ ] Docker containers building
- [ ] GitHub Actions CI/CD passing
- [ ] Environment variables documented
- [ ] Rollback procedures tested

---

## Risk Mitigation

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| DB migration issues | Medium | High | Test migrations locally first; version control all schema changes |
| Adapter integration complexity | Medium | High | Start with Claude (simplest); validate adapter interface early |
| Real-time WebSocket bugs | Low | Medium | Use proven library (Socket.io); extensive E2E testing |
| Performance at scale | Low | High | Profile early; use Redis caching; optimize DB indexes |
| Cost calculation errors | Medium | High | Unit test cost logic heavily; reconciliation job |
| VM hibernation timing | Low | Medium | Test wake-up latency; document cold-start behavior |
| GitHub Actions timeout | Low | Low | Keep test suites fast; split long tests across jobs |

---

## Known Constraints

1. **Adapters V1:** Only Claude, OpenClaw, Process (defer 6 others)
2. **DB Partitioning:** Defer heartbeat_run_events partitioning (post-MVP)
3. **Dev Mode:** Local executor at localhost:8080 (no Fly.io in dev)
4. **Frontend Scope:** All 11 pages, no scope reduction
5. **Tooling:** React Router + Vitest (not Jest)
6. **Clean-room:** No legacy v1 code reference

---

## Next Phase

**Phase 1: Monorepo Foundation** is ready to begin.

**Prerequisites:**
- Node 18+ installed
- pnpm installed
- GitHub repo initialized ✓

**Estimated completion:** 2 hours
**Owner:** Backend lead
**Blockers:** None

---

**Version:** 1.0
**Last Updated:** 2026-03-17
**Status:** All phases pending, ready to begin Phase 1
**Sync:** Updated from implementation plan (plans/260317-1118-blueprint-implementation/plan.md)
