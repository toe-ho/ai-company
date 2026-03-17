# Project Rebuild Documentation

Complete technical documentation for building an **AI Company Platform** — a system that lets anyone launch and run a fully autonomous AI-powered business from a web dashboard.

## What Is This Project?

A web platform where users create AI companies staffed entirely by AI agents. Users define a business goal, the platform assembles a team of AI agents (CEO, engineers, designers, marketers), and the agents work autonomously — writing code, designing, marketing, researching — 24/7.

**Target user:** Non-technical entrepreneurs who want to run businesses powered by AI, not manage infrastructure.

**Core pitch:** "Launch a full AI company for $2K/month instead of $250K/month."

## Documentation Index

### 1. Product & Vision — [01-product/](01-product/)

| # | File | Description |
|---|------|-------------|
| 00 | [Project Overview](01-product/00-project-overview.md) | What we're building, target users, tech stack |
| 01 | [Product Concept](01-product/01-product-concept.md) | Business model, user journeys, domain concepts |

### 2. AI System — [02-ai-system/](02-ai-system/)

| # | File | Description | Priority |
|---|------|-------------|----------|
| 02 | [AI Architecture](02-ai-system/02-ai-architecture.md) | Execution Engine + Adapter system | **High** |
| 03 | [AI Workflow](02-ai-system/03-ai-workflow.md) | Full 10-step agent execution lifecycle | **Critical** |
| 04 | [Agent Workflow](02-ai-system/04-agent-workflow.md) | Heartbeat loop, task checkout, self-hiring | **Critical** |
| 05 | [Prompt System](02-ai-system/05-prompt-system.md) | Skills, context injection, env vars | **High** |
| 06 | [Skill File Reference](02-ai-system/06-skill-file-reference.md) | Complete SKILL.md agent instruction doc | **Critical** |
| 07 | [Agent Executor Spec](02-ai-system/07-agent-executor-spec.md) | HTTP API, SSE format, process management | **Critical** |
| 08 | [Adapter Implementation Guide](02-ai-system/08-adapter-implementation-guide.md) | Per-adapter: command, args, output parsing | **Critical** |

### 3. System & Code Architecture — [03-architecture/](03-architecture/)

| # | File | Description | Priority |
|---|------|-------------|----------|
| 09 | [System Architecture](03-architecture/09-system-architecture.md) | Control plane + execution plane diagram | |
| 10 | [Data Flow](03-architecture/10-data-flow.md) | End-to-end request lifecycle diagrams | |
| 11 | [Backend Architecture](03-architecture/11-backend-architecture.md) | NestJS overview, clean architecture layers | |
| 12 | [API Architecture (NestJS)](03-architecture/12-api-architecture-nestjs.md) | Full CQRS structure, Drizzle schemas, modules | **Critical** |
| 12a | [Authentication Architecture](03-architecture/12a-auth-architecture.md) | Better Auth, guards, decorators, IActor (Phase 2) | **High** |
| 12b | [Auth Quick Reference](03-architecture/12b-auth-quick-reference.md) | Common auth patterns, usage examples | |
| 13 | [Frontend & UI](03-architecture/13-frontend-and-ui.md) | React app, folder structure, non-tech UX | |
| 14 | [Monorepo Setup Guide](03-architecture/14-monorepo-setup-guide.md) | Turborepo + pnpm, build order, Docker, CI/CD | |

### 4. Data & API — [04-data-and-api/](04-data-and-api/)

| # | File | Description | Priority |
|---|------|-------------|----------|
| 15 | [Database Design](04-data-and-api/15-database-design.md) | Multi-tenant schema, all tables | |
| 16 | [Database Migration Strategy](04-data-and-api/16-database-migration-strategy.md) | Drizzle migrations, partitioning, indexes | |
| 17 | [API Design](04-data-and-api/17-api-design.md) | All 50+ endpoints, auth, formats | |
| 18 | [API Response Schemas](04-data-and-api/18-api-response-schemas.md) | Exact JSON shapes for key endpoints | **Critical** |

### 5. Operations & Security — [05-operations/](05-operations/)

| # | File | Description |
|---|------|-------------|
| 19 | [Auth, Security & Permissions](05-operations/19-auth-security-and-permissions.md) | Multi-tenant auth, API key vault, governance |
| 20 | [Background Jobs & Async](05-operations/20-background-jobs-and-async-processing.md) | Scheduler, wakeups, VM lifecycle |
| 21 | [Error Handling Patterns](05-operations/21-error-handling-patterns.md) | Exception filter, domain exceptions |
| 22 | [Tools & Integrations](05-operations/22-tools-and-integrations.md) | Adapters, Fly.io, storage, external APIs |

### 6. Infrastructure & Testing — [06-infrastructure/](06-infrastructure/)

| # | File | Description |
|---|------|-------------|
| 23 | [Config & Environment](06-infrastructure/23-config-and-environment.md) | Env vars, Fly.io config |
| 24 | [Deployment & Infra](06-infrastructure/24-deployment-and-infra.md) | Cloud hosting, CI/CD, monitoring |
| 25 | [Testing Strategy](06-infrastructure/25-testing-strategy.md) | Unit, integration, E2E tests; CI pipeline |

## Recommended Reading Order

1. **Product:** `00` → `01`
2. **AI system:** `02` → `03` → `04` → `05` → `06` → `07` → `08`
3. **Architecture:** `09` → `11` → `12` → `13` → `14`
4. **Data:** `15` → `16` → `17` → `18`
5. **Security & ops:** `19` → `20` → `21`
6. **Deploy & test:** `23` → `24` → `25`
