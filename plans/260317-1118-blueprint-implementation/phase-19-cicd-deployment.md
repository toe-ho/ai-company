# Phase 19: CI/CD & Deployment

## Context Links
- [24 - Deployment & Infrastructure](../../docs/blueprint/06-infrastructure/24-deployment-and-infra.md)
- [14 - Monorepo Setup](../../docs/blueprint/03-architecture/14-monorepo-setup-guide.md) (CI section)
- [23 - Config & Environment](../../docs/blueprint/06-infrastructure/23-config-and-environment.md)

## Overview
- **Priority:** P3
- **Status:** pending
- **Effort:** 2h
- **Description:** GitHub Actions CI/CD workflows, Docker configs, Fly.io deployment configs, environment variable documentation.

## Key Insights
- Pipeline: typecheck -> unit tests -> build -> E2E tests -> deploy
- Turborepo cache speeds up CI (skip unchanged packages)
- Backend deploys to Railway or Fly.io
- Executor deploys as Fly.io Machine image
- Frontend deploys as static site (Vite build output)
- Multi-replica safe: pg advisory lock handles scheduler dedup

## Architecture
```
.github/workflows/
├── ci.yml                        # PR verification
├── deploy-backend.yml            # Deploy backend on main push
├── deploy-executor.yml           # Build + push executor Docker image
└── deploy-web.yml                # Deploy frontend

apps/backend/
├── Dockerfile
└── fly.toml

apps/executor/
├── Dockerfile                    # (Created in Phase 14)
└── fly.toml

apps/web/
└── Dockerfile                    # Optional (Vite static build)
```

## Related Code Files

### Files to Create
- `.github/workflows/ci.yml`
- `.github/workflows/deploy-backend.yml`
- `.github/workflows/deploy-executor.yml`
- `.github/workflows/deploy-web.yml`
- `apps/backend/Dockerfile`
- `apps/backend/fly.toml`
- `apps/executor/fly.toml`

## Implementation Steps

1. **Create CI workflow** (`.github/workflows/ci.yml`)
   ```yaml
   name: CI
   on: [pull_request]
   jobs:
     typecheck:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: pnpm/action-setup@v3
           with: { version: 9 }
         - uses: actions/setup-node@v4
           with: { node-version: 20, cache: 'pnpm' }
         - run: pnpm install --frozen-lockfile
         - run: turbo typecheck

     unit-tests:
       needs: typecheck
       runs-on: ubuntu-latest
       services:
         postgres:
           image: postgres:16
           env:
             POSTGRES_DB: test
             POSTGRES_USER: test
             POSTGRES_PASSWORD: test
           ports: ['5432:5432']
           options: >-
             --health-cmd pg_isready
             --health-interval 10s
             --health-timeout 5s
             --health-retries 5
       steps:
         - uses: actions/checkout@v4
         - uses: pnpm/action-setup@v3
           with: { version: 9 }
         - uses: actions/setup-node@v4
           with: { node-version: 20, cache: 'pnpm' }
         - run: pnpm install --frozen-lockfile
         - run: turbo test
           env:
             TEST_DATABASE_URL: postgres://test:test@localhost:5432/test

     build:
       needs: unit-tests
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: pnpm/action-setup@v3
           with: { version: 9 }
         - uses: actions/setup-node@v4
           with: { node-version: 20, cache: 'pnpm' }
         - run: pnpm install --frozen-lockfile
         - run: turbo build
   ```

2. **Create backend Dockerfile** (`apps/backend/Dockerfile`)
   ```dockerfile
   FROM node:20-slim AS base
   RUN npm install -g pnpm

   FROM base AS installer
   WORKDIR /app
   COPY pnpm-workspace.yaml package.json pnpm-lock.yaml turbo.json ./
   COPY packages/shared/ ./packages/shared/
   COPY apps/backend/ ./apps/backend/
   RUN pnpm install --frozen-lockfile

   FROM installer AS builder
   RUN pnpm --filter @ai-company/backend build

   FROM node:20-slim AS runner
   WORKDIR /app
   COPY --from=builder /app/apps/backend/dist ./dist
   COPY --from=builder /app/node_modules ./node_modules
   COPY --from=builder /app/apps/backend/node_modules ./apps/backend/node_modules

   EXPOSE 3100
   CMD ["node", "dist/main.js"]
   ```

3. **Create backend fly.toml** (`apps/backend/fly.toml`)
   ```toml
   app = "aico-backend"

   [build]
     dockerfile = "Dockerfile"

   [http_service]
     internal_port = 3100
     force_https = true

   [[http_service.checks]]
     interval = "30s"
     timeout = "5s"
     path = "/api/health"
     method = "GET"
   ```

4. **Create backend deploy workflow** (`.github/workflows/deploy-backend.yml`)
   ```yaml
   name: Deploy Backend
   on:
     push:
       branches: [main]
       paths: ['apps/backend/**', 'packages/shared/**']
   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: superfly/flyctl-actions/setup-flyctl@master
         - run: flyctl deploy --config apps/backend/fly.toml
           env:
             FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
   ```

5. **Create executor fly.toml** (`apps/executor/fly.toml`)
   ```toml
   app = "aico-executor"

   [build]
     dockerfile = "Dockerfile"

   [http_service]
     internal_port = 8080

   [mounts]
     source = "workspace_data"
     destination = "/workspace"
   ```

6. **Create executor deploy workflow** (`.github/workflows/deploy-executor.yml`)
   - Build Docker image
   - Push to Fly.io registry
   - This is the template image; actual VMs are created by FlyioProvisionerService

7. **Create web deploy workflow** (`.github/workflows/deploy-web.yml`)
   ```yaml
   name: Deploy Web
   on:
     push:
       branches: [main]
       paths: ['apps/web/**', 'packages/shared/**']
   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: pnpm/action-setup@v3
           with: { version: 9 }
         - uses: actions/setup-node@v4
           with: { node-version: 20, cache: 'pnpm' }
         - run: pnpm install --frozen-lockfile
         - run: pnpm --filter @ai-company/web build
         # Deploy static build (Cloudflare Pages, Vercel, or Fly.io)
         - run: echo "Deploy apps/web/dist to hosting provider"
   ```

8. **Document required secrets** in repository settings:
   - `FLY_API_TOKEN`: Fly.io deploy token
   - `DATABASE_URL`: Production database URL (for migrations)
   - `TEST_ANTHROPIC_KEY`: For E2E tests
   - `TURBO_TOKEN` + `TURBO_TEAM`: Turborepo remote cache (optional)

9. **Create migration run script**
   - Add to deploy workflow: run migrations before health check
   - `pnpm --filter @ai-company/backend drizzle-kit migrate`

## Todo List
- [ ] CI workflow (typecheck -> test -> build)
- [ ] Backend Dockerfile
- [ ] Backend fly.toml
- [ ] Backend deploy workflow
- [ ] Executor fly.toml
- [ ] Executor deploy workflow
- [ ] Web deploy workflow
- [ ] Document required secrets
- [ ] Migration script in deploy pipeline
- [ ] Verify CI passes on clean branch

## Success Criteria
- CI runs on every PR: typecheck -> test -> build
- Backend deploys to Fly.io on main push
- Executor image builds and pushes to registry
- Frontend builds as static site
- Health check passes post-deploy
- Migrations run automatically on deploy

## Risk Assessment
- **First deploy:** May need manual Fly.io app creation + secrets setup
- **Database migrations in deploy:** Run before new code starts; use rolling deploy
- **Turborepo cache in CI:** Optional; works without but slower

## Security Considerations
- All secrets in GitHub Actions secrets (never in code)
- Fly.io deploy token scoped to specific apps
- DATABASE_URL only used in migration step (not exposed to runtime)
- E2E test key is a separate low-privilege key

## Next Steps
- Project is fully deployable. Post-launch: monitoring, performance tuning, feature iteration.

---

## Unresolved Questions (Across All Phases)

1. **Event sourcing for audit:** Blueprint mentions activityLog for audit, but should heartbeat events use event sourcing pattern for full replay capability? V1: No, simple table.
2. **Shared package publishing:** Internal workspace only, or publish to npm for external consumers? V1: Internal only.
3. **Docker multi-stage for backend:** Should backend Dockerfile also pre-install agent CLIs for local dev testing? Probably no -- executor handles that.
4. **Drizzle seed scripts:** Where should seed data live? Recommend `apps/backend/src/infrastructure/persistence/seeds/`.
5. **Better Auth version pinning:** 1.5.5 specifically called out in blueprint. Need to verify compatibility with NestJS setup.
6. **Fly.io region strategy:** Default `sjc` -- should users be able to choose region? Blueprint says yes (runnerConfig.region).
7. **WebSocket auth:** RealtimeGateway should validate session cookie on connection. Implementation detail not fully specified in blueprint.
8. **Rate limiting:** Not mentioned in blueprint. Add basic rate limiting (express-rate-limit) on auth endpoints for V1?
9. **Frontend E2E test API key:** Need a test/sandbox Anthropic key for CI. Or mock the validation endpoint in E2E.
