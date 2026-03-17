# 14 — Monorepo Setup Guide

Turborepo + pnpm workspaces setup for the AI Company Platform.

## Package Manager & Build System

| Tool | Version | Purpose |
|------|---------|---------|
| pnpm | 9+ | Package manager + workspaces |
| Turborepo | 2+ | Build orchestration + caching |
| Node.js | 20+ | Runtime |
| TypeScript | 5+ | Language (all packages) |

## Repository Layout

```
your-product/
├── apps/
│   ├── backend/            ← NestJS API + Scheduler (CQRS, @nestjs/schedule)
│   ├── web/                ← React frontend (Vite)
│   └── executor/           ← Agent Executor (Fly.io VM)
├── packages/
│   ├── shared/             ← Types, constants, validators (Zod)
│   ├── adapters/           ← Agent runtime integrations
│   └── adapter-utils/      ← Shared adapter utilities
├── config/
│   ├── skills/             ← Agent instruction files
│   └── templates/          ← Company templates
├── tests/                  ← E2E tests (Playwright)
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

## pnpm Workspace Config

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```json
// package.json (root)
{
  "private": true,
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "typecheck": "turbo typecheck",
    "lint": "turbo lint",
    "db:migrate": "turbo db:migrate --filter=@your-product/backend"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.0.0"
  }
}
```

## turbo.json Configuration

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"]
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "typecheck": {
      "dependsOn": ["^typecheck"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "db:migrate": {
      "cache": false
    }
  }
}
```

## Package Dependency Graph

```
packages/shared          ← no internal deps (pure TS)
    ↑
packages/adapter-utils   ← depends on shared
    ↑
packages/adapters        ← depends on shared + adapter-utils
    ↑
apps/backend             ← depends on shared + @nestjs/schedule
apps/web                 ← depends on shared
apps/executor            ← depends on shared + adapters + adapter-utils
```

**Build order enforced by Turborepo:** `shared` → `adapter-utils` → `adapters` → `backend` / `web` / `executor`

## Package Configurations

### packages/shared

```json
// packages/shared/package.json
{
  "name": "@your-product/shared",
  "version": "0.0.1",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  }
}
```

Contains: entity interfaces, enums, Zod schemas, API path constants. No runtime dependencies — TypeScript only.

### packages/adapter-utils

```json
{
  "name": "@your-product/adapter-utils",
  "dependencies": {
    "@your-product/shared": "workspace:*"
  }
}
```

### packages/adapters

```json
{
  "name": "@your-product/adapters",
  "dependencies": {
    "@your-product/shared": "workspace:*",
    "@your-product/adapter-utils": "workspace:*"
  }
}
```

### apps/backend

```json
{
  "name": "@your-product/backend",
  "dependencies": {
    "@your-product/shared": "workspace:*",
    "@nestjs/core": "^10.0.0",
    "@nestjs/cqrs": "^10.0.0",
    "@nestjs/schedule": "^4.0.0",
    "drizzle-orm": "^0.38.0",
    "drizzle-kit": "^0.30.0"
  }
}
```

### apps/executor

```json
{
  "name": "@your-product/executor",
  "dependencies": {
    "@your-product/shared": "workspace:*",
    "@your-product/adapters": "workspace:*",
    "@your-product/adapter-utils": "workspace:*"
  }
}
```

## Dev Workflow

```bash
# Install all dependencies
pnpm install

# Run backend + web in parallel (turbo handles ordering)
turbo dev

# Run only the backend
turbo dev --filter=@your-product/backend

# Run only the web app
turbo dev --filter=@your-product/web

# Build everything (respects dependency order)
turbo build

# Type-check all packages
turbo typecheck

# Run all tests
turbo test

# Add a dependency to a specific app
pnpm --filter @your-product/backend add some-package
```

`turbo dev` starts `apps/backend` and `apps/web` in parallel. Turborepo ensures `packages/shared` is compiled first (since both depend on it).

## Docker Build for Executor (Fly.io VM Image)

The executor runs inside Fly.io VMs as a Docker container. Build targets only `apps/executor` and its workspace dependencies:

```dockerfile
# apps/executor/Dockerfile
FROM node:20-slim AS base
RUN npm install -g pnpm

# Copy workspace files needed for the executor
FROM base AS installer
WORKDIR /app
COPY pnpm-workspace.yaml package.json turbo.json ./
COPY packages/shared/ ./packages/shared/
COPY packages/adapter-utils/ ./packages/adapter-utils/
COPY packages/adapters/ ./packages/adapters/
COPY apps/executor/ ./apps/executor/
RUN pnpm install --frozen-lockfile

FROM installer AS builder
RUN pnpm --filter @your-product/executor build

FROM base AS runner
WORKDIR /app
# Copy only built output + node_modules
COPY --from=builder /app/apps/executor/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Install agent CLIs (Claude, etc.)
RUN npm install -g @anthropic-ai/claude-code

EXPOSE 3200
CMD ["node", "dist/main.js"]
```

Deploy the executor image to Fly.io as a Machine template. The `FlyioProvisionerService` boots instances of this image per company.

## Environment Variable Management

Environment variables are **not shared** across packages via Turborepo — each app manages its own `.env` file.

```
apps/backend/.env       ← DATABASE_URL, REDIS_URL, JWT_SECRET, FLYIO_API_TOKEN, etc.
apps/web/.env           ← VITE_API_BASE_URL
apps/executor/.env      ← injected at runtime by provisioner (not committed)
```

Use `.env.example` files (committed) to document required variables. Never commit `.env` files.

For CI/CD secrets, inject via environment variables in the pipeline — not via `.env` files.

The `packages/shared` package does **not** read env vars. Config parsing happens only in `apps/backend/src/config/` using NestJS `ConfigModule`.

## CI/CD Pipeline with Turborepo Cache

Turborepo remote cache (via Vercel or self-hosted) speeds up CI by skipping unchanged packages.

```yaml
# .github/workflows/ci.yml (excerpt)
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile

      # Turborepo remote cache
      - run: turbo typecheck
        env: { TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}, TURBO_TEAM: ${{ vars.TURBO_TEAM }} }

      - run: turbo test
        env: { TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}, TURBO_TEAM: ${{ vars.TURBO_TEAM }} }

      - run: turbo build
        env: { TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}, TURBO_TEAM: ${{ vars.TURBO_TEAM }} }

      # Deploy (only on main branch)
      - if: github.ref == 'refs/heads/main'
        run: |
          turbo build --filter=@your-product/backend
          flyctl deploy --config apps/backend/fly.toml
```

Pipeline order: `typecheck → test → build → deploy`

With Turborepo cache, unchanged packages are skipped entirely — only affected packages are rebuilt and retested.
