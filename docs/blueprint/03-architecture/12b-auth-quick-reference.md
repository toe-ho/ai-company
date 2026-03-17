# 12b — Authentication Quick Reference

Fast lookup for common auth patterns in Phase 2.

## Guard Application

### Global Protection (All Routes by Default)
```typescript
// Already applied globally via APP_GUARD
// BoardAuthGuard protects all routes unless @AllowAnonymous
```

### Skip Protection for Public Endpoints
```typescript
@Post('/auth/sign-in')
@AllowAnonymous()
async signIn(@Body() dto: SignInDto) { ... }
```

### Protect Agent Endpoints
```typescript
@UseGuards(AgentAuthGuard, CompanyAccessGuard)
@Patch('/issues/:id')
async updateIssue(
  @Param('id') issueId: string,
  @CurrentActor() actor: IActor,
  @RunId() runId: string,
) { ... }
```

### Enforce Role Requirements
```typescript
@UseGuards(CompanyRoleGuard)
@Roles('owner', 'admin')
@Post('/agents')
async createAgent(
  @CurrentActor() actor: IActor,
  @Body() dto: CreateAgentDto,
) { ... }
```

## Accessing Request Context

### Current Actor
```typescript
constructor(
  @Inject('IIssueRepository')
  private readonly issueRepository: IIssueRepository,
) {}

async create(
  @CurrentActor() actor: IActor,
) {
  // actor.type = 'board' | 'agent'
  // actor.userId = string (board only)
  // actor.agentId = string (agent only)
  // actor.companyId = string
  // actor.runId = string (agent only)
}
```

### Company ID
```typescript
@Get(':companyId/dashboard')
async getDashboard(
  @CompanyId() companyId: string,
  @CurrentActor() actor: IActor,
) {
  // companyId from route param
  // or falls back to actor.companyId
}
```

### Run ID (Agent Mutations)
```typescript
@Patch('/issues/:id')
async update(
  @RunId() runId: string,
  @Body() dto: UpdateIssueDto,
) {
  // runId from X-Run-Id header
  // Enables audit trail linking mutations to specific heartbeat
}
```

## Authentication Methods

### Session (Board Users)
```
User logs in via /api/auth/sign-in
→ Better Auth creates session
→ Cookie returned to browser
→ Subsequent requests include cookie
→ BoardAuthGuard validates via auth.api.getSession()
```

### Bearer JWT (Agent Ephemeral)
```
Heartbeat execution generates JWT:
  { agentId, companyId, runId, exp: +48h }

Agent uses in callback:
  Authorization: Bearer <jwt>

AgentAuthGuard verifies signature & expiry
```

### Persistent API Key (Agent Long-Lived)
```
Created: POST /api/agents/:id/keys
Format: pcp_<32-char random>
Stored as: SHA-256 hash

Agent uses in callback:
  Authorization: pcp_XXXX

AgentAuthGuard hashes & compares against DB
```

## Common Patterns

### Verify Company Membership (Board User)
```typescript
@UseGuards(CompanyAccessGuard)
@Get(':companyId/agents')
async listAgents(@CompanyId() companyId: string) { ... }
```
Guard checks UserCompanyModel for membership.

### Verify Agent in Company
```typescript
@UseGuards(AgentAuthGuard, CompanyAccessGuard)
@Post('/issues/:id/checkout')
async checkout(
  @CurrentActor() actor: IActor,
) { ... }
```
AgentAuthGuard sets actor from JWT/key; CompanyAccessGuard verifies resource belongs to actor.companyId.

### Verify Role + Company
```typescript
@UseGuards(CompanyRoleGuard)
@Roles('owner', 'admin')
@Post(':companyId/billing')
async updateBilling(
  @CurrentActor() actor: IActor,
  @CompanyId() companyId: string,
) { ... }
```
CompanyRoleGuard queries UserCompanyModel for role in companyId.

## Error Responses

### Invalid Session (BoardAuthGuard)
```json
{
  "statusCode": 401,
  "message": "Valid session required",
  "error": "Unauthorized"
}
```

### Invalid JWT/API Key (AgentAuthGuard)
```json
{
  "statusCode": 401,
  "message": "Invalid authorization scheme",
  "error": "Unauthorized"
}
```

### No Company Access (CompanyAccessGuard)
```json
{
  "statusCode": 403,
  "message": "Forbidden",
  "error": "Forbidden"
}
```

### Insufficient Role (CompanyRoleGuard)
```json
{
  "statusCode": 403,
  "message": "Forbidden",
  "error": "Forbidden"
}
```

## Configuration Files

### env.example
```bash
AUTH_SECRET=<32-char-random>
AUTH_URL=http://localhost:3000
AUTH_GOOGLE_CLIENT_ID=<oauth-id>
AUTH_GOOGLE_CLIENT_SECRET=<oauth-secret>
AUTH_GITHUB_CLIENT_ID=<oauth-id>
AUTH_GITHUB_CLIENT_SECRET=<oauth-secret>
```

### auth.config.ts
```typescript
export const authConfig = {
  secret: process.env.AUTH_SECRET,
  url: process.env.AUTH_URL,
  googleClientId: process.env.AUTH_GOOGLE_CLIENT_ID,
  googleClientSecret: process.env.AUTH_GOOGLE_CLIENT_SECRET,
  githubClientId: process.env.AUTH_GITHUB_CLIENT_ID,
  githubClientSecret: process.env.AUTH_GITHUB_CLIENT_SECRET,
};
```

## Testing Auth

### Mock Session (Board User)
```typescript
const mockActor: IActor = {
  type: ActorType.Board,
  userId: 'user-123',
};
// Set in request.actor before controller call
```

### Mock JWT (Agent)
```typescript
const token = agentJwtService.sign({
  agentId: 'agent-456',
  companyId: 'company-789',
  runId: 'run-000',
  exp: Date.now() + 48 * 60 * 60 * 1000,
});
// Use in Authorization: Bearer <token>
```

### Mock API Key
```typescript
const key = 'pcp_' + randomBytes(32).toString('hex');
const hash = hashApiKey(key);
// Insert into agent_api_keys table before test
// Use in Authorization: <key>
```

## See Also

- **[12a — Authentication Architecture](12a-auth-architecture.md)** — Full auth details
- **[19 — Auth, Security & Permissions](../05-operations/19-auth-security-and-permissions.md)** — Security policies
- **[11 — Backend Architecture](11-backend-architecture.md)** — Overall NestJS structure
