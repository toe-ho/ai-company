# 12a — Authentication Architecture (Phase 2)

NestJS authentication & authorization layer using Better Auth 1.5.5, guards, and decorators.

## Overview

Phase 2 introduced a complete authentication system protecting all routes via global guard pattern. Supports dual authentication methods:
- **Board (human users):** Session-based via Better Auth
- **Agents:** Bearer JWT or persistent API keys

## Better Auth Integration

### Configuration
- **Library:** better-auth 1.5.5
- **Database:** Dedicated pg.Pool (not Drizzle)
- **Tables:** `users`, `sessions`, `accounts`, `verification`
- **Session TTL:** 30 days (configurable)
- **Field Mapping:** camelCase → snake_case

Example mapping:
```typescript
user: {
  modelName: 'users',
  fields: {
    emailVerified: 'email_verified',
    createdAt: 'created_at',
    updatedAt: 'updated_at',
  }
},
session: {
  modelName: 'sessions',
  fields: {
    userId: 'user_id',
    expiresAt: 'expires_at',
    ipAddress: 'ip_address',
    userAgent: 'user_agent',
  }
},
account: {
  modelName: 'accounts',
  fields: {
    accountId: 'account_id',
    providerId: 'provider_id',
    userId: 'user_id',
    accessToken: 'access_token',
    refreshToken: 'refresh_token',
    idToken: 'id_token',
  }
}
```

### Supported Auth Methods
- Email/password with auto sign-in
- OAuth: Google, GitHub
- Cookie-based sessions

## AuthService

```typescript
// application/services/impl/auth-service.ts
@Injectable()
export class AuthService implements OnModuleInit {
  auth!: ReturnType<typeof betterAuth>;

  onModuleInit(): void {
    const pool = new Pool({ connectionString: dbUrl });
    this.auth = betterAuth({ database: pool, ... });
  }
}
```

**Responsibilities:**
- Wraps Better Auth instance with dedicated pool
- Provides `auth.api.getSession()` for guards
- Manages user creation, OAuth flows
- Handles email verification

## AgentJwtService

```typescript
// application/services/impl/agent-jwt-service.ts
@Injectable()
export class AgentJwtService {
  sign(payload: {
    agentId: string;
    companyId: string;
    runId: string;
    exp: number;
  }): string

  verify(token: string): JwtPayload
}
```

**Responsibilities:**
- Create ephemeral JWT (48h TTL) per heartbeat
- Verify JWT claims in guards
- Track agent identity and execution context

## Guard Architecture

### Guard Hierarchy

```
Every Request
    ↓
[BoardAuthGuard] (APP_GUARD - global default)
    ↓ (unless @AllowAnonymous)
Sets request.actor from session
    ↓
[AgentAuthGuard] (if @UseGuards applied)
    ↓
Validates Bearer JWT or pcp_ API key
    ↓
[CompanyAccessGuard] (if @UseGuards applied)
    ↓
Verifies actor.companyId ∈ resource
    ↓
[CompanyRoleGuard] (if @UseGuards applied)
    ↓
Validates actor role ∈ @Roles
    ↓
Handler executes
```

### BoardAuthGuard

Protects human user endpoints. Registered globally via `APP_GUARD`.

```typescript
@Injectable()
export class BoardAuthGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const isAnon = reflector.getAllAndOverride(ALLOW_ANONYMOUS_KEY, [...]);
    if (isAnon) return true;

    const headers = fromNodeHeaders(request.headers);
    const session = await authService.auth.api
      .getSession({ headers })
      .catch(() => null);

    if (!session?.user?.id) {
      throw new UnauthorizedException('Valid session required');
    }

    request.actor = {
      type: ActorType.Board,
      userId: session.user.id,
    };
    return true;
  }
}
```

**Behavior:**
- Skipped if route has `@AllowAnonymous()`
- Validates session cookie
- Sets `request.actor` with user ID
- Throws 401 if invalid/expired

### AgentAuthGuard

Protects agent callback endpoints. Validates two auth schemes.

```typescript
@Injectable()
export class AgentAuthGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const authHeader = request.headers['authorization'];

    if (raw.startsWith('Bearer ')) {
      return this.handleJwt(raw.slice(7), request);
    }
    if (raw.startsWith('pcp_')) {
      return this.handleApiKey(raw, request);
    }
    throw new UnauthorizedException('Invalid authorization scheme');
  }

  private handleJwt(token: string, request: { actor?: IActor }): boolean {
    const payload = agentJwtService.verify(token);
    request.actor = {
      type: ActorType.Agent,
      agentId: payload.agentId,
      companyId: payload.companyId,
      runId: payload.runId,
    };
    return true;
  }

  private async handleApiKey(key: string, request: { actor?: IActor }): Promise<boolean> {
    const hash = hashApiKey(key);
    const apiKeyRecord = await apiKeyRepo.findOne({ where: { hash } });
    if (!apiKeyRecord) throw new UnauthorizedException('Invalid API key');

    request.actor = {
      type: ActorType.Agent,
      agentId: apiKeyRecord.agentId,
      companyId: apiKeyRecord.companyId,
    };
    return true;
  }
}
```

**Behavior:**
- Accepts Bearer JWT or pcp_-prefixed keys
- Verifies JWT signature and expiry
- Hashes and compares API keys against DB
- Sets `request.actor` with agent context

### CompanyAccessGuard

Validates actor belongs to requested company.

```typescript
@UseGuards(CompanyAccessGuard)
@Get(':companyId/agents')
async listAgents(@CompanyId() companyId: string) { ... }
```

**Behavior:**
- Compares `actor.companyId` with route company
- Verifies user has membership (via UserCompanyModel)
- Throws 403 if access denied

### CompanyRoleGuard

Enforces role-based access control.

```typescript
@UseGuards(CompanyRoleGuard)
@Roles('owner', 'admin')
@Post(':companyId/agents')
async createAgent(@Body() dto: CreateAgentDto) { ... }
```

**Behavior:**
- Extracts roles from @Roles() decorator
- Queries UserCompanyModel for user role
- Throws 403 if role not in allowed set

## Decorators

All decorators defined in `decorator/` directory.

### @AllowAnonymous()
```typescript
@AllowAnonymous()
@Post('/auth/sign-up')
async signUp(@Body() dto: SignUpDto) { ... }
```
- Metadata key: `ALLOW_ANONYMOUS_KEY`
- Skips BoardAuthGuard check
- Used for login, signup, public endpoints

### @CurrentActor()
```typescript
@Post('/tasks')
async create(
  @CurrentActor() actor: IActor,
  @Body() dto: CreateTaskDto
) { ... }
```
- Extracts `request.actor`
- Type: `IActor`

### @CompanyId()
```typescript
@Get(':companyId/dashboard')
async getDashboard(@CompanyId() companyId: string) { ... }
```
- Extracts company UUID from route params
- Falls back to `actor.companyId` if not in route
- Type: `string` (UUID)

### @RunId()
```typescript
@Patch(':id')
async update(
  @RunId() runId: string,
  @Body() dto: UpdateDto
) { ... }
```
- Extracts `X-Run-Id` header
- Links all agent mutations to specific heartbeat run
- Type: `string` (UUID)

### @Roles(...)
```typescript
@Roles('owner', 'admin')
@Post('/invite-user')
async inviteUser(@Body() dto: InviteDto) { ... }
```
- Metadata key: `ROLES_KEY`
- Requires CompanyRoleGuard
- Values: `'owner'`, `'admin'`, `'viewer'`

## IActor Interface

```typescript
interface IActor {
  type: ActorType;        // 'board' | 'agent' | 'system'
  userId?: string;        // Board users only
  agentId?: string;       // Agent actors only
  companyId?: string;     // All types
  runId?: string;         // Agent actors only
}
```

**Set by guards:**

| Guard | type | userId | agentId | companyId | runId |
|-------|------|--------|---------|-----------|-------|
| BoardAuthGuard | board | ✓ | - | - | - |
| AgentAuthGuard (JWT) | agent | - | ✓ | ✓ | ✓ |
| AgentAuthGuard (key) | agent | - | ✓ | ✓ | - |

## AuthModule

```typescript
// auth/auth-module.ts
@Module({
  imports: [DrizzleModule],
  controllers: [AuthController],
  providers: [
    AuthService,
    AgentJwtService,
    {
      provide: USER_COMPANY_REPOSITORY,
      useClass: UserCompanyRepository,
    },
    AgentAuthGuard,
    CompanyAccessGuard,
    CompanyRoleGuard,
    {
      provide: APP_GUARD,
      useClass: BoardAuthGuard,
    },
  ],
  exports: [
    AuthService,
    AgentJwtService,
    AgentAuthGuard,
    CompanyAccessGuard,
    CompanyRoleGuard,
    USER_COMPANY_REPOSITORY,
  ],
})
export class AuthModule {}
```

## API Key Management

### Persistent Agent API Keys

Created via `POST /api/agents/:id/keys`:

```typescript
// utils/hash.ts
export function hashApiKey(key: string): string {
  return crypto.createHash('sha256').update(key).digest('hex');
}
```

**Storage (agent_api_keys table):**
- Format: `pcp_<32-char random>`
- Stored as: SHA-256 hash
- Returned once to user on creation
- No expiry, revocable

**Validation in AgentAuthGuard:**
1. Extract `pcp_` key from Authorization header
2. Hash with SHA-256
3. Query agent_api_keys table for matching hash
4. If found, extract agentId + companyId
5. Set request.actor and continue

## Usage Examples

### Login Endpoint
```typescript
@Controller('auth')
export class AuthController {
  @Post('sign-in')
  @AllowAnonymous()
  async signIn(@Body() dto: SignInDto) {
    return this.authService.auth.api.signInEmail(dto);
  }

  @Post('sign-up')
  @AllowAnonymous()
  async signUp(@Body() dto: SignUpDto) {
    return this.authService.auth.api.signUpEmail(dto);
  }
}
```

### Protected Board Endpoint
```typescript
@Controller('companies')
export class BoardCompanyController {
  @Post()
  @UseGuards(CompanyAccessGuard)
  async create(
    @CurrentActor() actor: IActor,
    @Body() dto: CreateCompanyDto
  ) {
    return this.commandBus.execute(
      new CreateCompanyCommand(actor.userId, dto)
    );
  }
}
```

### Protected Agent Endpoint
```typescript
@Controller('issues')
export class AgentIssueController {
  @Post(':id/checkout')
  @UseGuards(AgentAuthGuard, CompanyAccessGuard)
  async checkout(
    @Param('id') issueId: string,
    @CurrentActor() actor: IActor,
    @RunId() runId: string,
    @Body() dto: CheckoutIssueDto
  ) {
    return this.commandBus.execute(
      new CheckoutIssueCommand(issueId, actor.agentId, actor.companyId, runId, dto)
    );
  }
}
```

## Security Considerations

- **Never log auth headers or tokens** — scrubbed by HttpLoggerInterceptor
- **API keys stored as hash** — plaintext shown once only on creation
- **Dedicated pool for Better Auth** — isolates session management from business queries
- **JWT claims include companyId + runId** — enables server to validate agent scope
- **BoardAuthGuard is default** — opt-in to bypass with @AllowAnonymous, not the other way around
