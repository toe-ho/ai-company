# Phase 6: Auth Layer

## Context Links
- [12a - Auth Architecture](../../docs/blueprint/03-architecture/12a-auth-architecture.md)
- [12b - Auth Quick Reference](../../docs/blueprint/03-architecture/12b-auth-quick-reference.md)
- [19 - Auth Security](../../docs/blueprint/05-operations/19-auth-security-and-permissions.md)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 3h
- **Description:** Complete auth system: Better Auth integration, AuthService, AgentJwtService, 4 guards, 5 decorators, AuthModule.

## Key Insights
- Better Auth uses its own `pg.Pool` (not Drizzle) for session management
- BoardAuthGuard is global (APP_GUARD); all routes protected by default
- @AllowAnonymous bypasses BoardAuthGuard for public endpoints
- AgentAuthGuard supports two schemes: Bearer JWT (ephemeral, 48h) and pcp_ API keys (persistent)
- Guard execution order: BoardAuth -> AgentAuth -> CompanyAccess -> CompanyRole
- `request.actor` set by guards, consumed by @CurrentActor decorator

## Requirements
### Functional
- Better Auth email/password + OAuth (Google, GitHub)
- Session-based auth for board users (cookie)
- JWT auth for agents (per-heartbeat, 48h TTL)
- Persistent API key auth for agents (pcp_ prefix, SHA-256 hashed)
- Company access verification
- Role-based access (owner/admin/viewer)

### Non-functional
- Dedicated pg.Pool for Better Auth (isolated from Drizzle)
- Never log auth headers or tokens
- API keys returned once on creation, never again

## Architecture
```
apps/backend/src/
├── auth/
│   └── auth-module.ts
├── guard/
│   ├── board-auth-guard.ts
│   ├── agent-auth-guard.ts
│   ├── company-access-guard.ts
│   └── company-role-guard.ts
├── decorator/
│   ├── allow-anonymous.ts
│   ├── current-actor.ts
│   ├── company-id.ts
│   ├── run-id.ts
│   └── roles.ts
├── application/services/
│   ├── interface/
│   │   ├── auth-service.interface.ts
│   │   └── agent-jwt-service.interface.ts
│   └── impl/
│       ├── auth-service.ts
│       └── agent-jwt-service.ts
└── utils/
    └── hash.ts
```

## Related Code Files

### Files to Create
- `apps/backend/src/auth/auth-module.ts`
- `apps/backend/src/guard/board-auth-guard.ts`
- `apps/backend/src/guard/agent-auth-guard.ts`
- `apps/backend/src/guard/company-access-guard.ts`
- `apps/backend/src/guard/company-role-guard.ts`
- `apps/backend/src/decorator/allow-anonymous.ts`
- `apps/backend/src/decorator/current-actor.ts`
- `apps/backend/src/decorator/company-id.ts`
- `apps/backend/src/decorator/run-id.ts`
- `apps/backend/src/decorator/roles.ts`
- `apps/backend/src/application/services/impl/auth-service.ts`
- `apps/backend/src/application/services/impl/agent-jwt-service.ts`
- `apps/backend/src/utils/hash.ts`

## Implementation Steps

1. **Create `utils/hash.ts`**
   ```typescript
   import crypto from 'node:crypto';
   export function hashApiKey(key: string): string {
     return crypto.createHash('sha256').update(key).digest('hex');
   }
   ```

2. **Create `AuthService`**
   - Implements `OnModuleInit`
   - Creates dedicated `pg.Pool` with DATABASE_URL
   - Initializes `betterAuth({ database: pool, ... })` with field mappings per blueprint 12a
   - Exposes `auth.api.getSession()` for guards
   - Supports email/password with autoSignIn

3. **Create `AgentJwtService`**
   - `sign(payload: { agentId, companyId, runId, exp })`: returns JWT string using `jsonwebtoken`
   - `verify(token: string)`: returns decoded payload or throws
   - Uses `AGENT_JWT_SECRET` from ConfigService
   - Default TTL: 48h (`AGENT_JWT_TTL_SECONDS`)

4. **Create decorators**
   - `@AllowAnonymous()`: `SetMetadata('ALLOW_ANONYMOUS', true)`
   - `@CurrentActor()`: `createParamDecorator` extracting `request.actor`
   - `@CompanyId()`: Extract from route params `:companyId` or `:cid`, fallback to `actor.companyId`
   - `@RunId()`: Extract `X-Run-Id` header
   - `@Roles(...roles)`: `SetMetadata('ROLES', roles)`

5. **Create `BoardAuthGuard`**
   - Implements `CanActivate`
   - Check `@AllowAnonymous` metadata -> skip if true
   - Call `authService.auth.api.getSession({ headers: fromNodeHeaders(req.headers) })`
   - If no session -> throw `UnauthorizedException`
   - Set `request.actor = { type: 'board', userId: session.user.id }`

6. **Create `AgentAuthGuard`**
   - Extract `Authorization` header
   - If starts with `Bearer ` -> verify JWT via AgentJwtService -> set actor with agentId, companyId, runId
   - If starts with `pcp_` -> hash with SHA-256 -> lookup in agent_api_keys table -> set actor with agentId, companyId
   - Otherwise -> throw `UnauthorizedException`

7. **Create `CompanyAccessGuard`**
   - For board actors: query `userCompanies` table for (userId, companyId) membership
   - For agent actors: verify `actor.companyId` matches route companyId
   - Throw `ForbiddenException` if no access

8. **Create `CompanyRoleGuard`**
   - Read `@Roles` metadata
   - If no roles specified -> pass (no restriction)
   - Query `userCompanies` for user's role in company
   - Throw `ForbiddenException` if role not in allowed set

9. **Create `AuthModule`**
   - Providers: AuthService, AgentJwtService, all guards, UserCompanyRepository
   - `{ provide: APP_GUARD, useClass: BoardAuthGuard }` -> global default
   - Exports: AuthService, AgentJwtService, guards, UserCompanyRepository

## Todo List
- [ ] hash utility
- [ ] AuthService (Better Auth integration)
- [ ] AgentJwtService (JWT sign/verify)
- [ ] 5 decorators (@AllowAnonymous, @CurrentActor, @CompanyId, @RunId, @Roles)
- [ ] BoardAuthGuard (global APP_GUARD)
- [ ] AgentAuthGuard (JWT + API key)
- [ ] CompanyAccessGuard
- [ ] CompanyRoleGuard
- [ ] AuthModule
- [ ] typecheck passes

## Success Criteria
- Public routes accessible with @AllowAnonymous
- Board routes require valid session cookie
- Agent routes accept both JWT and pcp_ API keys
- Company access verified for all scoped routes
- Role-based access enforced where @Roles applied

## Risk Assessment
- **Better Auth version:** Pin to 1.5.5; API may change in newer versions
- **Dedicated pool leak:** Ensure pool cleanup on module destroy (implement `OnModuleDestroy`)
- **JWT secret rotation:** Not handled in V1; document as future improvement

## Security Considerations
- Never log Authorization headers (HttpLoggerInterceptor must scrub)
- API keys stored as SHA-256 hash only
- JWT includes companyId + runId for scope validation
- Better Auth sessions have 30-day TTL
- BoardAuthGuard is opt-out (not opt-in) -- secure by default

## Next Steps
- Phase 7: Core services (use AuthService for session management)
- Phase 11: Guards registered via AuthModule imported by AppModule
