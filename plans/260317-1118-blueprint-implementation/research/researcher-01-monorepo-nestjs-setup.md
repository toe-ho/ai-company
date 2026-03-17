# Production-Grade Monorepo Setup: NestJS + Drizzle + pnpm/Turborepo

## 1. NestJS + Drizzle + CQRS Architecture

### Module Structure Pattern
- **Controllers → Services → Repositories → Drizzle Client**: Standard NestJS flow
- **Repository Pattern**: Drizzle doesn't natively include repositories; wrap client methods per entity to lock to specific types
- **DatabaseModule**: Use `forRoot()` pattern with connection pool as provider for async config
- **Global Registration**: Export DatabaseModule from root, then inject DrizzleConnection into services

### CQRS Handler Patterns
- **Commands**: Create CommandHandler classes implementing `ICommandHandler<CommandName>`
- **Queries**: Create QueryHandler classes implementing `IQueryHandler<QueryName>`
- **Command/Query Objects**: Plain classes containing only payload data
- **Handler Registration**: Register in feature module via `CommandBus.register()` and `QueryBus.register()`
- **Separation**: Commands modify state through Drizzle transactions; Queries read directly via repositories

### Best Practice for Drizzle Integration
- Create feature-specific repository classes (e.g., `UserRepository`, `PostRepository`)
- Repositories: Encapsulate Drizzle table queries, expose business logic methods
- Services: Depend on repositories, implement business logic, invoke commands/queries
- Handlers: Call services to execute domain logic
- **Key Win**: Drizzle's query builder is typed, eliminating runtime type errors vs raw SQL

## 2. pnpm Workspaces + Turborepo 2 Configuration

### Directory Structure
```
monorepo/
├── pnpm-workspace.yaml          # Root: paths: ["apps/*", "packages/*"]
├── turbo.json                   # Build pipeline config
├── apps/
│   ├── backend/                 # NestJS app
│   └── web/                      # React + Vite app
└── packages/
    ├── database/                # Drizzle schema + migrations
    ├── types/                   # Shared TypeScript types
    └── auth/                    # Better Auth config
```

### Turborepo Configuration (turbo.json)
```json
{
  "tasks": {
    "build": {
      "outputs": ["dist/**", ".next/**"],
      "cache": true,
      "dependsOn": ["^build"]
    },
    "dev": {
      "cache": false,
      "dependsOn": ["^build"]
    },
    "test": {
      "outputs": ["coverage/**"],
      "cache": true
    }
  }
}
```

### Key Patterns
- **pnpm-workspace.yaml**: Lists workspace paths; pnpm handles phantom dependencies automatically
- **Dependency Declarations**: Use `workspace:*` in package.json for internal packages
- **Caching Benefit**: Initial build ~30s → cache hit ~0.2s via Turborepo
- **Task Dependencies**: `dependsOn: ["^build"]` means "run build on dependencies first"

## 3. Better Auth + NestJS Integration

### Setup Pattern
- **Package**: Use `@thallesp/nestjs-better-auth` (community-maintained, robust)
- **Module Registration**: `BetterAuthModule.forRoot({ ... })` in AppModule
- **Global Guard**: AuthGuard registered by default (can disable and use `@UseGuards()` manually)
- **Session Injection**: `@Session() session: Session` decorator in controllers

### Session Access in Routes
```typescript
@Get('/profile')
profile(@Session() session: Session) {
  return { user: session.user, isAuthenticated: !!session };
}
```

### Key Routes to Handle
- **Raw Requests**: Better Auth needs raw body access; disable NestJS's body parser globally
- **Auth Endpoints**: Delegate to Better Auth's built-in `/api/auth/*` routes
- **Optional Auth**: Use `@OptionalAuth()` decorator for public routes that may have user context
- **Admin Routes**: Custom guards layering on top of AuthGuard for role-based access

### Database Integration
- Better Auth generates session/verification tables automatically
- Store these in `packages/database` alongside your business schema
- Sessions table: Indexed on userId, sessionToken for fast lookups

## 4. Drizzle ORM Migration Strategy (35+ Tables)

### Schema Organization
- **Multi-file Structure**: Split schema into logical groups:
  - `packages/database/schema/user.ts` (users, profiles)
  - `packages/database/schema/auth.ts` (sessions, tokens)
  - `packages/database/schema/business.ts` (domain tables)
- **Drizzle Kit Config**: Use glob pattern to discover all schemas
  ```js
  schema: ['./schema/**/*.ts']
  ```

### Migration Workflow
1. **Development**: `drizzle-kit push` → Direct schema push (no migration files)
2. **Production**:
   - `drizzle-kit generate` → Create migration file in `drizzle/migrations/`
   - Review SQL for safety (avoid downtime on large tables)
   - Apply migrations: `node -r tsx ./drizzle-kit migrate` or use Neon console

### Safe Deployment Patterns
- **Additive DDL**: Always safe (new columns, tables, indexes)
- **Risky Operations**: Column removals, renames, type changes on large tables
- **Strategy**: Use dual-write pattern for renames; add new column, deprecate old, remove in later release
- **Transaction Handling**: All migrations run in single transaction; rollback on any error

### Neon-Specific Notes
- Use `DATABASE_URL` environment variable (no SSL certs needed for Neon)
- Migrations execute directly against your database
- Connection pooling handled by Neon's serverless driver

## Unresolved Questions

1. **State Management for CQRS Events**: Should event sourcing be used for audit trail, or just logs?
2. **Monorepo Publishing**: Will shared packages be published to npm, or remain internal?
3. **Docker Strategy**: Will each workspace get its own Dockerfile, or shared multi-stage build?
4. **Drizzle Seeding**: Should seed scripts live in packages/database or apps/backend?

---

**Sources:**
- [Best ORM for NestJS in 2025: Drizzle ORM vs TypeORM vs Prisma](https://dev.to/sasithwarnakafonseka/best-orm-for-nestjs-in-2025-drizzle-orm-vs-typeorm-vs-prisma-229c)
- [2025 NestJS + React 19 + Drizzle ORM + Turborepo Architecture ADR](https://dev.to/xiunotes/2025-nestjs-react-19-drizzle-orm-turborepo-architecture-decision-record-3o1k)
- [Repository Pattern in Nest.js with Drizzle ORM](https://medium.com/@vimulatus/repository-pattern-in-nest-js-with-drizzle-orm-e848aa75ecae)
- [pnpm Workspaces](https://pnpm.io/workspaces)
- [Better Auth NestJS Integration](https://better-auth.com/docs/integrations/nestjs)
- [Drizzle ORM Migrations Guide](https://orm.drizzle.team/docs/migrations)
- [Neon + Drizzle Migrations](https://neon.com/docs/guides/drizzle-migrations)
- [Drizzle Migration Patterns](https://medium.com/@bhagyarana80/8-drizzle-orm-patterns-for-clean-fast-migrations-456c4c35b9d8)
