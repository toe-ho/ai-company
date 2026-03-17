# Phase 1: Monorepo Foundation

## Context Links
- [14 - Monorepo Setup Guide](../../docs/blueprint/03-architecture/14-monorepo-setup-guide.md)
- [Research: Monorepo + NestJS Setup](research/researcher-01-monorepo-nestjs-setup.md)

## Overview
- **Priority:** P1 (everything depends on this)
- **Status:** pending
- **Effort:** 2h
- **Description:** Bootstrap pnpm workspaces + Turborepo monorepo with all 6 packages, TypeScript configs, tooling.

## Key Insights
- Build order: `shared` -> `adapter-utils` -> `adapters` -> `backend` / `web` / `executor`
- `packages/shared` has zero runtime deps (pure TS)
- Each app manages its own `.env`; `packages/shared` never reads env vars
- Turborepo v2 uses `tasks` key (not `pipeline`)

## Requirements
### Functional
- All 6 packages buildable independently
- `turbo dev` starts backend + web in parallel
- `turbo build` respects dependency order
- `turbo typecheck` checks all packages

### Non-functional
- Node.js 20+, pnpm 9+, TypeScript 5+
- Turborepo remote cache ready (env var based)

## Architecture
```
ai-orchestration-company-v2/
├── apps/
│   ├── backend/          <- NestJS API + Scheduler
│   ├── web/              <- React + Vite frontend
│   └── executor/         <- Agent Executor (Express, Fly.io)
├── packages/
│   ├── shared/           <- Types, enums, Zod validators, constants
│   ├── adapters/         <- Agent runtime integrations
│   └── adapter-utils/    <- Shared adapter utilities
├── config/
│   ├── skills/           <- Agent instruction files (placeholder)
│   └── templates/        <- Company templates (placeholder)
├── tests/                <- E2E tests (Playwright)
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── .gitignore
├── .prettierrc
└── .eslintrc.cjs
```

## Related Code Files

### Files to Create
- `package.json` (root)
- `pnpm-workspace.yaml`
- `turbo.json`
- `tsconfig.base.json`
- `.gitignore`
- `.prettierrc`
- `.eslintrc.cjs`
- `.env.example`
- `apps/backend/package.json`
- `apps/backend/tsconfig.json`
- `apps/backend/tsconfig.build.json`
- `apps/backend/nest-cli.json`
- `apps/backend/.env.example`
- `apps/web/package.json`
- `apps/web/tsconfig.json`
- `apps/web/tsconfig.app.json`
- `apps/web/vite.config.ts`
- `apps/web/index.html`
- `apps/web/.env.example`
- `apps/executor/package.json`
- `apps/executor/tsconfig.json`
- `packages/shared/package.json`
- `packages/shared/tsconfig.json`
- `packages/shared/src/index.ts`
- `packages/adapters/package.json`
- `packages/adapters/tsconfig.json`
- `packages/adapters/src/index.ts`
- `packages/adapter-utils/package.json`
- `packages/adapter-utils/tsconfig.json`
- `packages/adapter-utils/src/index.ts`
- `config/skills/.gitkeep`
- `config/templates/.gitkeep`
- `tests/.gitkeep`

## Implementation Steps

1. **Create root `package.json`**
   - `private: true`
   - Scripts: `dev`, `build`, `test`, `typecheck`, `lint`, `db:migrate`
   - DevDeps: `turbo@^2`, `typescript@^5`, `prettier`, `eslint`

2. **Create `pnpm-workspace.yaml`**
   ```yaml
   packages:
     - "apps/*"
     - "packages/*"
   ```

3. **Create `turbo.json`**
   - `tasks.build`: `dependsOn: ["^build"]`, outputs: `["dist/**"]`
   - `tasks.dev`: `cache: false`, `persistent: true`, `dependsOn: ["^build"]`
   - `tasks.typecheck`: `dependsOn: ["^typecheck"]`
   - `tasks.test`: `dependsOn: ["^build"]`, outputs: `["coverage/**"]`
   - `tasks.lint`: `outputs: []`
   - `tasks["db:migrate"]`: `cache: false`

4. **Create `tsconfig.base.json`**
   - Target: `ES2022`, module: `Node16`, strict: true
   - `paths` aliases not needed (workspace protocol handles it)

5. **Create shared package** (`packages/shared`)
   - `name: "@ai-company/shared"`, main/types point to `./src/index.ts`
   - Barrel export in `src/index.ts`
   - Zero dependencies

6. **Create adapter-utils package** (`packages/adapter-utils`)
   - `name: "@ai-company/adapter-utils"`, depends on `@ai-company/shared: workspace:*`

7. **Create adapters package** (`packages/adapters`)
   - `name: "@ai-company/adapters"`, depends on `@ai-company/shared` + `@ai-company/adapter-utils`

8. **Create backend app** (`apps/backend`)
   - `name: "@ai-company/backend"`, depends on `@ai-company/shared`
   - NestJS deps: `@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express`, `@nestjs/cqrs`, `@nestjs/schedule`, `@nestjs/config`
   - Drizzle deps: `drizzle-orm`, `drizzle-kit`, `postgres` (driver)
   - Other: `zod`, `better-auth`, `ioredis`, `pino`, `pino-http`, `jsonwebtoken`
   - Dev: `@nestjs/cli`, `@nestjs/testing`, `vitest`, `@vitest/coverage-v8`
   <!-- Updated: Validation Session 1 - Vitest instead of Jest -->
   - `nest-cli.json` with standard config
   - `.env.example` with all required vars from blueprint 23

9. **Create web app** (`apps/web`)
   - `name: "@ai-company/web"`, depends on `@ai-company/shared`
   - Deps: `react@19`, `react-dom@19`, `react-router`, `@tanstack/react-query`, `tailwindcss@4`, `lucide-react`
   - Dev: `vite`, `@vitejs/plugin-react`, `typescript`, `vitest`
   - Routing: `react-router@7`
   <!-- Updated: Validation Session 1 - React Router 7 + Vitest -->
   - `vite.config.ts` with React plugin, proxy `/api` to backend
   - `index.html` shell

10. **Create executor app** (`apps/executor`)
    - `name: "@ai-company/executor"`, depends on `@ai-company/shared` + `@ai-company/adapters` + `@ai-company/adapter-utils`
    - Deps: `express`, `@types/express`

11. **Create config dirs** with `.gitkeep` placeholders

12. **Create `.gitignore`**
    - node_modules, dist, .env, coverage, .turbo, .next

13. **Create `.prettierrc`** and `.eslintrc.cjs`
    - Standard NestJS + React config

14. **Run `pnpm install`** and verify
15. **Run `turbo typecheck`** to verify all packages resolve

## Todo List
- [ ] Root package.json, pnpm-workspace.yaml, turbo.json
- [ ] tsconfig.base.json
- [ ] packages/shared skeleton
- [ ] packages/adapter-utils skeleton
- [ ] packages/adapters skeleton
- [ ] apps/backend skeleton with NestJS deps
- [ ] apps/web skeleton with Vite + React
- [ ] apps/executor skeleton
- [ ] config/ and tests/ dirs
- [ ] .gitignore, .prettierrc, .eslintrc.cjs
- [ ] .env.example files
- [ ] pnpm install + turbo typecheck passes

## Success Criteria
- `pnpm install` completes without errors
- `turbo typecheck` passes for all packages
- `turbo build` completes (empty builds OK)
- Workspace dependency graph is correct (`pnpm ls --filter @ai-company/backend` shows `@ai-company/shared`)

## Risk Assessment
- **pnpm version mismatch:** Pin pnpm version in `packageManager` field
- **Turborepo v2 config:** Use `tasks` not `pipeline` key
- **TypeScript path resolution:** Use `workspace:*` protocol, not tsconfig paths

## Security Considerations
- `.env` files in `.gitignore`
- `.env.example` files contain only variable names, no values
- No secrets committed

## Next Steps
- Phase 2: Populate `packages/shared` with types, enums, Zod schemas
