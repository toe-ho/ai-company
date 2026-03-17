# 19 — Auth, Security & Permissions

## User Authentication

### Better Auth (Session-Based)
- `POST /api/auth/sign-up/email` — Create account
- `POST /api/auth/sign-in/email` — Login
- OAuth support: Google, GitHub
- Cookie-based sessions stored in `authSessions` table
- Session expiry + refresh

### User → Company Relationship
- Users can own/access multiple companies
- `userCompanies` junction table with role
- Roles: `owner`, `admin`, `viewer`
- Owner can manage billing, delete company
- Admin can manage agents, approve requests
- Viewer can only watch dashboard

## Agent Authentication

### Ephemeral JWT (Per Heartbeat)
Created by server during execution:
```
Claims: { sub: agentId, company_id, run_id, exp: +48h }
```
- Injected as `API_KEY` env var into the Fly.io VM
- Used by agent to call back to control plane API
- 48-hour TTL (configurable)

### Persistent Agent API Key
- **Creation:** `POST /api/agents/:id/keys`
- **Format:** `pcp_<random>` (shown once only)
- **Storage:** SHA-256 hash in `agent_api_keys` table
- **Expiry:** None (revocable via `DELETE /api/agents/:id/keys/:keyId`)
- **Validation:** `hashApiKey()` utility compares provided key against stored hash
- **Use Case:** Agents needing long-lived access without heartbeat JWT

### X-Run-Id Header (Audit Trail)
All agent mutations must include:
```
X-Run-Id: <heartbeat-run-uuid>
```
Links every API call to a specific execution for traceability.

## LLM API Key Security (Critical)

Users enter their LLM API keys (Anthropic, OpenAI, Google) in the dashboard. These keys are the most sensitive data in the system.

### Security Flow
```
1. User pastes key in UI
2. Client sends to server via HTTPS
3. Server validates key (test API call)
4. Server encrypts with AES-256
5. Stored in companyApiKeys table (encrypted)
6. Raw key NEVER stored, NEVER logged
7. During heartbeat: decrypted in server memory only
8. Passed to Execution Engine as part of execution request
9. Agent Executor injects as env var to agent process on VM
10. After heartbeat: key reference discarded from memory
```

### What We NEVER Do
- Store raw API keys anywhere
- Log API keys in activity log or events
- Write keys to VM filesystem
- Include keys in run events or transcripts
- Send keys to browser after initial entry
- Store keys in Redis or cache

### Key Validation
On entry, we make a test API call:
```
Anthropic: GET /v1/models (with key)
  → 200: Valid ✅
  → 401: Invalid key → show error with guidance
  → 429: Rate limited → valid but throttled ⚠️
```

## Authentication Implementation (Phase 2)

### Better Auth Integration
- **Service:** `AuthService` wraps `betterAuth()`
- **Database:** Dedicated `pg.Pool` (not Drizzle)
- **Tables:** `users`, `sessions`, `accounts`, `verification` (snake_case fields)
- **Session Expiry:** 30 days (configurable)
- **Field Mapping:** All camelCase defaults mapped to snake_case (e.g., `emailVerified` → `email_verified`)

### Guards (Global & Per-Route)

| Guard | Purpose | Auth Method |
|-------|---------|-------------|
| `BoardAuthGuard` | Default protection (APP_GUARD) | Session cookie via Better Auth |
| `AgentAuthGuard` | Agent callback endpoints | Bearer JWT or pcp_ API key |
| `CompanyAccessGuard` | Verify actor in company | Checks actor.companyId in resource |
| `CompanyRoleGuard` | Role-based access | Enforces @Roles() decorator |

**Usage:**
```typescript
@Post('/agents')
@Roles('owner', 'admin')  // Requires owner or admin role
async createAgent(@Body() dto: CreateAgentDto) { ... }

@Post('/auth/sign-in')
@AllowAnonymous()  // Bypass BoardAuthGuard
async signIn(@Body() dto: SignInDto) { ... }
```

### Decorators

| Decorator | Extracts | Example |
|-----------|----------|---------|
| `@CurrentActor()` | Request actor | `constructor(actor: IActor)` |
| `@CompanyId()` | Company UUID | `async create(@CompanyId() companyId: string)` |
| `@RunId()` | X-Run-Id header | `async update(@RunId() runId: string)` |
| `@AllowAnonymous()` | Skip BoardAuthGuard | `@AllowAnonymous()` |
| `@Roles(...)` | Require roles | `@Roles('owner', 'admin')` |

## Actor Model

Every request resolves to one of three actor types via `IActor`:

```typescript
interface IActor {
  type: ActorType;          // "board" | "agent" | "system"
  userId?: string;          // User ID (board actors only)
  agentId?: string;         // Agent ID (agent actors only)
  companyId?: string;       // Company UUID (all types)
  runId?: string;           // Heartbeat run UUID (agent actors only)
}
```

Set by guards:
- **BoardAuthGuard:** type = "board", userId set from session
- **AgentAuthGuard:** type = "agent", agentId/companyId from JWT payload

### Authorization Checks
- **Company access:** User must be member. Agent must be in same company.
- **Role-based:** Owner/admin can mutate. Viewer is read-only.
- **Agent scope:** Agents can only access their own company.

## Governance Controls

### Board Approval Gates
- Agent hiring requires approval (configurable)
- CEO strategy requires approval (configurable)
- Budget increases require approval

### Budget Enforcement
- Per-agent monthly budget (hard cap)
- Per-company monthly budget (hard cap)
- Auto-pause at 100%
- Warning alerts at 50%, 80%
- Kill switch: pause all agents instantly

### Agent Permissions
```json
{
  "agents:create": false,
  "tasks:assign": true,
  "tasks:assign_scope": "own_team",
  "approvals:create": true
}
```

### Self-Hiring Limits
- Max agents per company (default: 20)
- Max hire depth (default: 3 levels)
- Budget required on every hire
- Board approval required (default)

## Multi-Tenant Data Isolation

- Every table has `companyId`
- Every query filters by `companyId`
- Compound indexes for performance
- Users only see companies they belong to
- Agents only access their own company
- Fly.io VMs are per-company (hardware isolation via Firecracker)

## Input Validation

- Zod schemas validate all request bodies
- Invalid input → 400 with field-level errors
- Prevents injection, malformed data, type confusion

## Log Redaction

- API keys scrubbed from all logs
- Email addresses redacted in activity details
- Home paths replaced with `[redacted]` in agent output
- Adapter configs sanitized before logging

## Infrastructure Security

| Layer | Protection |
|-------|-----------|
| HTTPS | All traffic encrypted in transit |
| Fly.io VMs | Firecracker hardware isolation per company |
| Database | Managed PostgreSQL with encryption at rest |
| API Keys | AES-256 encryption at rest |
| Redis | TLS connection, no persistent storage |
| S3 | Server-side encryption |
| Secrets | Never in git, env vars for server config |
