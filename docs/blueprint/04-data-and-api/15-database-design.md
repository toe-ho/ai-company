# 15 — Database Design

## Overview

- **ORM:** Drizzle ORM (type-safe, PostgreSQL)
- **Multi-tenant:** All tables have `companyId`, every query scoped
- **Hosting:** Neon or Supabase managed PostgreSQL
- **Tables:** 35+

## Conceptual ER Diagram

```
users (auth) ──────────────────────────────────────────
  ├── sessions                                         │
  ├── userCompanies (junction)                         │
  └── billingAccounts                                  │
                                                       │
companies ─────────────────────────────────────────────┤
  │                                                    │
  ├── agents (reportsTo → agents.id)                   │
  │     ├── agentApiKeys                               │
  │     ├── agentRuntimeState                          │
  │     ├── agentTaskSessions                          │
  │     ├── agentConfigRevisions                       │
  │     ├── heartbeatRuns → heartbeatRunEvents          │
  │     └── agentWakeupRequests                        │
  │                                                    │
  ├── companyApiKeys (user's LLM keys, encrypted)      │
  │                                                    │
  ├── companyVms (Fly.io machine tracking)             │
  │                                                    │
  ├── projects → projectWorkspaces                     │
  │                                                    │
  ├── issues (parentId → issues.id)                    │
  │     ├── issueComments                              │
  │     ├── issueAttachments → assets                  │
  │     ├── issueLabels → labels                       │
  │     └── issueApprovals → approvals                 │
  │                                                    │
  ├── goals (parentId → goals.id)                      │
  │                                                    │
  ├── approvals → approvalComments                     │
  ├── costEvents                                       │
  ├── activityLog                                      │
  └── companyTemplates                                 │
```

## Core Tables

### users
| Column | Type | Notes |
|--------|------|-------|
| id | text | PK |
| name | text | Display name |
| email | text | Unique |
| emailVerified | boolean | |
| plan | text | "free" / "pro" / "enterprise" |
| createdAt | timestamp | |

### companies
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| ownerId | text | FK → users |
| name | text | Company name |
| description | text | |
| status | text | active / paused / archived |
| issuePrefix | text | UNIQUE, e.g. "APP" |
| issueCounter | integer | Auto-increment |
| budgetMonthlyCents | integer | Monthly cap |
| spentMonthlyCents | integer | Current spend |
| runnerConfig | jsonb | Fly.io VM config (region, size, etc.) |
| templateId | text | Source template |
| brandColor | text | UI theme |
| createdAt, updatedAt | timestamp | |

### agents
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| companyId | UUID | FK → companies |
| name | text | Agent name |
| role | text | ceo/cto/engineer/designer/pm/qa/marketer/etc. |
| title | text | Job title |
| icon | text | Avatar |
| status | text | active/paused/idle/running/error/terminated |
| reportsTo | UUID | FK → agents.id (org tree) |
| adapterType | text | claude/codex/openclaw_gateway/etc. |
| adapterConfig | jsonb | Model, env, timeout, etc. |
| runtimeConfig | jsonb | Heartbeat interval, max concurrent |
| budgetMonthlyCents | integer | Per-agent budget |
| spentMonthlyCents | integer | Per-agent spend |
| permissions | jsonb | What agent can do |
| lastHeartbeatAt | timestamp | |
| **Indexes** | | (company_id, status), (company_id, reports_to) |

### issues (tasks)
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| companyId | UUID | FK → companies |
| projectId | UUID | FK → projects |
| goalId | UUID | FK → goals |
| parentId | UUID | FK → issues.id (hierarchical) |
| title | text | |
| description | text | Markdown |
| status | text | backlog/todo/in_progress/in_review/done/blocked/cancelled |
| priority | text | critical/high/medium/low |
| assigneeAgentId | UUID | FK → agents (single assignee) |
| checkoutRunId | UUID | FK → heartbeatRuns |
| identifier | text | UNIQUE, e.g. "APP-42" |
| issueNumber | integer | Sequential per company |
| **Indexes** | | (company_id, status), (company_id, assignee_agent_id, status), UNIQUE(identifier) |

### heartbeatRuns
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| companyId | UUID | FK → companies |
| agentId | UUID | FK → agents |
| vmMachineId | text | Fly.io machine ID |
| invocationSource | text | timer/assignment/on_demand/automation |
| status | text | queued/running/succeeded/failed/cancelled/timed_out |
| startedAt, finishedAt | timestamp | |
| exitCode | integer | |
| usageJson | jsonb | Token counts |
| resultJson | jsonb | Execution result |
| computeCostCents | integer | Fly.io compute cost |
| stdoutExcerpt | text | Last output lines |
| **Index** | | (company_id, agent_id, started_at) |

### heartbeatRunEvents
| Column | Type | Notes |
|--------|------|-------|
| id | bigserial | PK |
| runId | UUID | FK → heartbeatRuns |
| seq | integer | Event sequence |
| eventType | text | |
| stream | text | system/stdout/stderr |
| message | text | |
| payload | jsonb | |
| **Indexes** | | (run_id, seq) |

### companyApiKeys (user's LLM keys)
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| companyId | UUID | FK → companies |
| provider | text | "anthropic" / "openai" / "google" |
| encryptedKey | text | AES-256 encrypted |
| keyHash | text | For dedup/lookup |
| label | text | User-friendly name |
| isValid | boolean | Last validation result |
| lastValidatedAt | timestamp | |
| lastUsedAt | timestamp | |
| createdAt | timestamp | |

### companyVms (Fly.io machine tracking)
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| companyId | UUID | FK → companies, UNIQUE |
| machineId | text | Fly.io machine ID |
| status | text | stopped/starting/running/hibernating |
| region | text | Fly.io region |
| size | text | CPU/RAM config |
| volumeId | text | Persistent storage |
| lastActiveAt | timestamp | For idle detection |
| createdAt | timestamp | |

### companyTemplates
| Column | Type | Notes |
|--------|------|-------|
| id | text | PK (slug) |
| name | text | "AI SaaS Startup" |
| description | text | |
| category | text | "tech" / "marketing" / "agency" |
| agentConfigs | jsonb | Pre-configured agent definitions |
| goalTemplate | text | Default company goal |
| isPublic | boolean | Available to all users |
| createdAt | timestamp | |

## Supporting Tables

### Costs & Billing
- **costEvents** — Per-event: agentId, provider, model, inputTokens, outputTokens, costCents, computeCostCents
- **billingAccounts** — User billing: plan, stripeCustomerId, credits
- **usageRecords** — Monthly rollup per company

### Activity & Comments
- **activityLog** — Immutable audit: actorType, action, entityType, entityId, details
- **issueComments** — Author: agent or user, markdown content
- **approvalComments** — Comments on approval requests

### Access Control
- **userCompanies** — Junction: userId ↔ companyId with role
- **agentApiKeys** — Hashed internal API keys for agent auth (not user's LLM keys)

### Other
- **goals** — Hierarchical (parentId), levels: company/team/agent/task
- **projects** — Linked to goals, has workspaces
- **projectWorkspaces** — Git repos, working directories
- **labels** — Custom tags for issues
- **approvals** — Type: hire_agent/approve_strategy, status machine
- **assets** — File storage metadata (S3 object keys)
- **agentConfigRevisions** — Immutable config history
- **agentRuntimeState** — Current state + cumulative token counters
- **agentTaskSessions** — Per-task session persistence
- **agentWakeupRequests** — Wakeup queue with coalescing

## Key Design Patterns

### Multi-Tenant Isolation
All tables have `companyId`. Compound indexes on `(company_id, ...)`. Every query filters by company.

### JSONB for Flexibility
`adapterConfig`, `runtimeConfig`, `runnerConfig`, `usageJson` — all JSONB. Add new provider? New config shape, zero migration.

### Soft Deletes
`archivedAt` on projects, `hiddenAt` on issues, `revokedAt` on keys.

### Hierarchies
`parentId` on issues and goals (self-referential). `reportsTo` on agents (org tree).

### Token Accounting
`agentRuntimeState` tracks cumulative totals. `costEvents` tracks per-event. `heartbeatRuns.usageJson` snapshots per-run.
