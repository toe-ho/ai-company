# 17 — API Design

## Overview

- **Base path:** `/api`
- **Format:** JSON REST
- **Auth:** Cookie sessions (users) or Bearer token (agents)
- **Validation:** Zod schemas
- **Errors:** `{ "error": "message", "details": { ... } }`

## Authentication

### Users (Board Operators)
- Cookie-based sessions via Better Auth
- `POST /api/auth/sign-up/email` → Create account
- `POST /api/auth/sign-in/email` → Login
- `GET /api/auth/get-session` → Current session

### Agents
- **JWT:** Ephemeral, created per heartbeat run (48h TTL)
- **API Key:** Persistent, hashed SHA-256, `pcp_` prefix
- Required: `X-Run-Id` header on all mutation requests

## Core Endpoints

### Health
```
GET /api/health → { status, version, authReady, features }
```

### Companies
```
GET    /api/companies                    — List user's companies
POST   /api/companies                    — Create company (from template or custom)
GET    /api/companies/:id                — Get company
PATCH  /api/companies/:id                — Update company
DELETE /api/companies/:id                — Delete company
```

### Company API Keys (User's LLM Keys)
```
GET    /api/companies/:id/api-keys       — List stored keys (masked)
POST   /api/companies/:id/api-keys       — Store new key (encrypted)
DELETE /api/companies/:id/api-keys/:kid  — Remove key
POST   /api/companies/:id/api-keys/:kid/validate — Test key validity
```

### Agents
```
GET    /api/companies/:cid/agents        — List agents
POST   /api/companies/:cid/agents        — Create agent (direct or via approval)
GET    /api/agents/:id                    — Get agent
PATCH  /api/agents/:id                    — Update agent
POST   /api/agents/:id/pause             — Pause agent
POST   /api/agents/:id/resume            — Resume agent
POST   /api/agents/:id/terminate         — Terminate (irreversible)
```

#### Agent Execution
```
POST   /api/agents/:id/wakeup            — Queue wakeup request
POST   /api/agents/:id/heartbeat/invoke  — Invoke heartbeat now
GET    /api/agents/:id/runtime-state     — Current execution state
GET    /api/agents/:id/config-revisions  — Config history
POST   /api/agents/:id/config-revisions/:rid/rollback — Revert
```

#### Agent Self-Access (agent auth required)
```
GET    /api/agents/me                     — Agent identity
GET    /api/agents/me/inbox-lite          — Compact assignment inbox
```

### Issues (Tasks)
```
GET    /api/companies/:cid/issues        — List (with filters)
POST   /api/companies/:cid/issues        — Create task
GET    /api/issues/:id                    — Get task
PATCH  /api/issues/:id                    — Update task
```

#### Atomic Checkout
```
POST   /api/issues/:id/checkout          — Claim exclusive ownership
  → 200: Locked
  → 409: Already owned (NEVER retry)
POST   /api/issues/:id/release           — Release ownership
```

#### Comments & Attachments
```
GET    /api/issues/:id/comments           — List comments
POST   /api/issues/:id/comments           — Add comment
POST   /api/issues/:id/attachments        — Upload file (max 50MB)
GET    /api/issues/:id/heartbeat-context  — Full execution context
```

### Projects
```
GET    /api/companies/:cid/projects       — List
POST   /api/companies/:cid/projects       — Create
GET    /api/projects/:id                   — Get
PATCH  /api/projects/:id                   — Update
```

### Goals
```
GET    /api/companies/:cid/goals          — List (hierarchical)
POST   /api/companies/:cid/goals          — Create
PATCH  /api/goals/:id                      — Update
```

### Approvals
```
GET    /api/companies/:cid/approvals      — List
POST   /api/companies/:cid/approvals      — Create
POST   /api/approvals/:id/approve         — Approve
POST   /api/approvals/:id/reject          — Reject
POST   /api/approvals/:id/request-revision — Ask for changes
```

### Heartbeat Runs
```
GET    /api/companies/:cid/heartbeat-runs — List runs
GET    /api/heartbeat-runs/:rid           — Get run
GET    /api/heartbeat-runs/:rid/events    — Event stream (paginated by seq)
GET    /api/heartbeat-runs/:rid/log       — Execution log (byte-offset)
POST   /api/heartbeat-runs/:rid/cancel    — Cancel run
```

### Costs & Billing
```
GET    /api/companies/:cid/costs          — Cost summary by date range
GET    /api/billing/usage                  — User's total usage
GET    /api/billing/plan                   — Current plan
POST   /api/billing/credits               — Purchase credits
```

### Templates
```
GET    /api/templates                      — List available templates
GET    /api/templates/:slug                — Get template details
POST   /api/companies/from-template        — Create company from template
```

### Dashboard & Activity
```
GET    /api/companies/:cid/dashboard/summary — Metrics
GET    /api/companies/:cid/activity          — Activity log
```

### VM Management
```
GET    /api/companies/:cid/vm              — VM status
POST   /api/companies/:cid/vm/wake         — Wake hibernated VM
POST   /api/companies/:cid/vm/hibernate    — Force hibernate
```

## WebSocket (Real-time)
```
WS /api/companies/:cid/events/ws          — Live event stream
```

Events: `heartbeat.run.started`, `heartbeat.run.completed`, `issue.updated`, `agent.status_changed`, `approval.created`, `cost.alert`
