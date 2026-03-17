---
title: "AI Company Platform v2 - Full Implementation"
description: "Greenfield build of NestJS + React platform where users create AI-powered companies with autonomous agent teams"
status: pending
priority: P1
effort: 49h
branch: main
tags: [greenfield, nestjs, react, cqrs, drizzle, fly-io, multi-tenant]
created: 2026-03-17
---

# AI Company Platform v2 - Implementation Plan

## Summary
Web platform where users create AI companies. Platform assembles AI agent teams (CEO, CTO, engineers) that work autonomously via heartbeat-driven execution on Fly.io VMs. ~400 files across 6 packages.

## Architecture
- **Control Plane:** NestJS API + Scheduler (CQRS, Clean Architecture, Drizzle ORM, PostgreSQL)
- **Execution Plane:** Fly.io VMs per company, Agent Executor (Express), 9 adapter types
- **Frontend:** React 19 + Vite + React Query + Tailwind CSS 4 + shadcn/ui
- **Monorepo:** pnpm workspaces + Turborepo, 6 packages

## Phase Overview

| # | Phase | Effort | Status | File |
|---|-------|--------|--------|------|
| 1 | Monorepo Foundation | 2h | pending | [phase-01](phase-01-monorepo-foundation.md) |
| 2 | Shared Package | 1h | pending | [phase-02](phase-02-shared-package.md) |
| 3 | Database Schema & Migrations | 3h | pending | [phase-03](phase-03-database-schema.md) |
| 4 | Domain Layer | 2h | pending | [phase-04](phase-04-domain-layer.md) |
| 5 | Infrastructure - Repositories | 3h | pending | [phase-05](phase-05-infrastructure-repositories.md) |
| 6 | Auth Layer | 3h | pending | [phase-06](phase-06-auth-layer.md) |
| 7 | Application - Core Services | 2h | pending | [phase-07](phase-07-core-services.md) |
| 8 | CQRS Handlers (Core) | 4h | pending | [phase-08](phase-08-cqrs-core.md) |
| 9 | CQRS Handlers (Supporting) | 3h | pending | [phase-09](phase-09-cqrs-supporting.md) |
| 10 | Domain Events | 2h | pending | [phase-10](phase-10-domain-events.md) |
| 11 | Cross-cutting & Modules | 2h | pending | [phase-11](phase-11-cross-cutting-modules.md) |
| 12 | Controllers & DTOs | 4h | pending | [phase-12](phase-12-controllers-dtos.md) |
| 13 | Real-time & Scheduler | 2h | pending | [phase-13](phase-13-realtime-scheduler.md) |
| 14 | Executor App | 3h | pending | [phase-14](phase-14-executor-app.md) |
| 15 | Adapters Package (3 of 9) | 2h | pending | [phase-15](phase-15-adapters-package.md) |
| 16 | Frontend Foundation | 3h | pending | [phase-16](phase-16-frontend-foundation.md) |
| 17 | Frontend Pages | 5h | pending | [phase-17](phase-17-frontend-pages.md) |
| 18 | Testing | 3h | pending | [phase-18](phase-18-testing.md) |
| 19 | CI/CD & Deployment | 2h | pending | [phase-19](phase-19-cicd-deployment.md) |

## Key Dependencies
```
Phase 1 (monorepo) -> Phase 2 (shared) -> Phase 3 (db) -> Phase 4 (domain) -> Phase 5 (repos)
Phase 5 + Phase 6 (auth) -> Phase 7 (services) -> Phase 8-9 (CQRS) -> Phase 10 (events)
Phase 10 + Phase 11 (modules) -> Phase 12 (controllers) -> Phase 13 (realtime)
Phase 2 -> Phase 14 (executor) -> Phase 15 (adapters)
Phase 2 -> Phase 16 (frontend) -> Phase 17 (pages)
Phase 12 + 17 -> Phase 18 (testing) -> Phase 19 (CI/CD)
```

## Blueprint Reference
All 27 docs in `docs/blueprint/` (01-product through 06-infrastructure).

## Validation Log

### Session 1 — 2026-03-17
**Trigger:** Initial plan creation validation before implementation
**Questions asked:** 8

#### Questions & Answers

1. **[Scope]** The plan builds all 9 adapter types (Claude, Codex, Cursor, Gemini, OpenCode, Pi, OpenClaw, Process, HTTP) in Phase 15. For V1 MVP, how many adapters should we implement?
   - Options: Claude + Process only | All 9 adapters | Claude + Codex + Process
   - **Answer:** Claude + Process + OpenClaw
   - **Custom input:** User selected Claude + Process + OpenClaw (custom combo)
   - **Rationale:** OpenClaw Gateway is architecturally different (WebSocket vs process spawn), validates that path. Claude is primary, Process is generic fallback.

2. **[Tooling]** Blueprint lists React Router for routing, but testing strategy says Jest while tech stack mentions Vitest. Which tools?
   - Options: React Router + Vitest | TanStack Router + Vitest | React Router + Jest
   - **Answer:** React Router + Vitest
   - **Rationale:** Vitest is faster, Vite-native, better ESM support. React Router is proven and matches blueprint.

3. **[Architecture]** The npm package scope is set to @ai-company/* in the plan. What scope name for monorepo packages?
   - Options: @ai-company/* | @paperclip/* | @ai-company/*
   - **Answer:** @ai-company/*
   - **Rationale:** Descriptive naming that matches the project concept. All package.json names must use @ai-company/ prefix.

4. **[Scope]** For V1 frontend, should we build all 11 pages or focus on critical path?
   - Options: Critical path only (6 pages) | All 11 pages | Backend first, minimal UI
   - **Answer:** All 11 pages
   - **Rationale:** User wants full UI from day one. No scope reduction on frontend.

5. **[Architecture]** Drizzle ORM driver: node-postgres vs @neondatabase/serverless?
   - Options: node-postgres (pg) | @neondatabase/serverless | Support both via config
   - **Answer:** node-postgres (pg)
   - **Rationale:** Standard driver, works everywhere, better for long-running NestJS server process. More portable.

6. **[Architecture]** For local dev without Fly.io, how to handle the Execution Engine?
   - Options: Local executor process | Mock execution engine | Docker Compose
   - **Answer:** Local executor process
   - **Rationale:** Run apps/executor locally, ExecutionEngine posts to localhost:8080. Real integration, no mocks.

7. **[Scope]** heartbeat_run_events table partitioning: implement in initial migration or defer?
   - Options: Defer to post-MVP | Implement from day one | Use Neon branching
   - **Answer:** Defer to post-MVP
   - **Rationale:** Simpler initial schema. Add partitioning when data volume warrants it.

8. **[Architecture]** Reference v1 project migrations or clean-room v2?
   - Options: Clean-room v2 | Review v1 migrations first
   - **Answer:** Clean-room v2
   - **Rationale:** Fresh start, reference blueprint docs only. No legacy code/schema constraints.

#### Confirmed Decisions
- **Adapters V1:** Claude + Process + OpenClaw only — saves ~1.5h, tests all 3 patterns (CLI spawn, generic, WebSocket)
- **Tooling:** React Router 7 + Vitest (not Jest) throughout
- **Package scope:** @ai-company/* (not @ai-company/*)
- **Frontend:** Full 11 pages, no scope reduction
- **DB Driver:** node-postgres (pg), not Neon serverless
- **Dev mode:** Local executor process at localhost:8080
- **Partitioning:** Defer heartbeat_run_events partitioning to post-MVP
- **Legacy:** Clean-room v2, no v1 code reference

#### Action Items
- [ ] Update all package.json names from @ai-company/* to @ai-company/*
- [ ] Update Phase 15 to only implement Claude + Process + OpenClaw adapters (defer 6 others)
- [ ] Update Phase 18 to use Vitest instead of Jest
- [ ] Update Phase 1 to specify React Router + Vitest in deps
- [ ] Update Phase 3 to use node-postgres driver explicitly, skip partitioning setup
- [ ] Update Phase 7 to include local dev mode for ExecutionEngineService
- [ ] Remove PartitionManagerWorker from Phase 13 (deferred)

#### Impact on Phases
- Phase 1: Change package scope from @ai-company/* to @ai-company/*, add vitest to deps
- Phase 3: Use pg driver, skip heartbeat_run_events partitioning, remove PartitionManagerWorker
- Phase 7: Add local dev mode to ExecutionEngineService (post to localhost:8080 when FLY_API_TOKEN not set)
- Phase 13: Remove PartitionManagerWorker (deferred to post-MVP)
- Phase 15: Reduce from 9 to 3 adapters (Claude + Process + OpenClaw). Effort: 3h → 2h
- Phase 16: Confirm React Router 7, add vitest config
- Phase 18: Replace Jest with Vitest throughout
