# 10 — Data Flow

## Flow 1: Non-Tech User Creates Company from Template

```
User (Browser)
    │
    ├─ Onboarding wizard: picks "AI SaaS Startup" template
    ├─ Enters goal: "Build a task management app"
    ├─ Adds API key: Anthropic key → validated → encrypted
    ├─ Clicks "Launch"
    │
    ▼
POST /api/companies/from-template
  { templateSlug: "saas-startup", goal: "...", apiKeyId: "..." }
    │
    ▼
Template Service
    ├─ Creates company record
    ├─ Creates agents from template (CEO, CTO, 2 engineers, QA)
    ├─ Creates company goal
    │
    ▼
Provisioner Service
    ├─ POST to Fly.io API → boot Machine (~3s)
    ├─ Attach persistent volume
    ├─ Store machineId in companyVms table
    │
    ▼
Heartbeat Service
    ├─ Creates initial task for CEO: "Define strategy for: {goal}"
    ├─ Enqueues CEO wakeup request
    │
    ▼
CEO Agent runs first heartbeat
    ├─ Reads goal
    ├─ Creates strategy as issues
    ├─ Assigns tasks to team
    ├─ Company is running!
    │
    ▼
Dashboard updates in real-time via WebSocket
```

## Flow 2: Agent Heartbeat Execution

```
Scheduler tick (every 30s)
    │
    ├─ Check each agent's heartbeat interval
    ├─ Engineer 1: lastHeartbeat + 300s < now → enqueue
    │
    ▼
Heartbeat Service
    ├─ Validate: active, under budget, no concurrent run
    ├─ Retrieve API key from vault (decrypt)
    │
    ▼
Provisioner
    ├─ Ensure VM running (wake if hibernated, ~3s)
    │
    ▼
Execution Engine
    ├─ POST to Agent Executor on Fly.io VM
    │  { runId, agent, adapter: "claude",
    │    apiKey: "sk-ant-...", workspace, context }
    │
    ▼
Agent Executor (on Fly.io VM)
    ├─ Inject ANTHROPIC_API_KEY as env var
    ├─ Symlink skill files
    ├─ Spawn: claude --prompt "..." --context-file session.json
    ├─ Stream stdout → SSE back to Control Plane
    │
    ▼
Agent Process
    ├─ GET /api/agents/me → identity
    ├─ GET /api/agents/me/inbox-lite → tasks
    ├─ POST /api/issues/:id/checkout → claim task
    ├─ GET /api/issues/:id/heartbeat-context → full context
    ├─ (writes code, runs tests, etc.)
    ├─ PATCH /api/issues/:id → status: "done"
    ├─ POST /api/issues/:id/comments → summary
    ├─ Exit code 0
    │
    ▼
Control Plane
    ├─ Parse result: tokens, cost, session
    ├─ Persist: heartbeatRuns, costEvents, agentRuntimeState
    ├─ Update agent.spentMonthlyCents
    ├─ Track Fly.io compute seconds
    ├─ Publish event → Redis → WebSocket → Dashboard
    │
    ▼
Idle timer starts → hibernate VM after 10 min
```

## Flow 3: Agent Requests to Hire

```
CEO Agent (during heartbeat)
    │
    ├─ POST /api/companies/:cid/approvals
    │  { type: "hire_agent", payload: { name: "Designer", role: "designer",
    │    adapterType: "openclaw_gateway", budget: 3000 } }
    │
    ▼
Approvals Service
    ├─ INSERT approvals (status: "pending")
    ├─ Publish event → "approval.created"
    │
    ▼
User sees notification in dashboard:
    "CEO wants to hire Designer (OpenClaw, $30/mo)"
    │
    ├─ [Approve]
    │
    ▼
POST /api/approvals/:id/approve
    │
    ▼
Hire Hook Service
    ├─ Create agent record
    ├─ Set reportsTo: ceoAgentId
    ├─ Generate internal API key
    ├─ Fly.io VM already running for company (reused)
    ├─ Agent starts receiving heartbeats
    │
    ▼
Designer agent is now active and working
```

## Flow 4: Cost Tracking

```
Agent process calls LLM API (via ANTHROPIC_API_KEY)
    │
    ▼
Adapter parses usage from agent output:
    { inputTokens: 85000, outputTokens: 12000, costUsd: 0.44 }
    │
    ▼
Heartbeat Service:
    ├─ INSERT costEvents
    │  { agentId, provider: "anthropic", model: "claude-sonnet",
    │    inputTokens: 85000, outputTokens: 12000, costCents: 44 }
    │
    ├─ UPDATE agentRuntimeState (cumulative totals)
    ├─ UPDATE agents SET spentMonthlyCents += 44
    ├─ Track Fly.io compute: computeCostCents += 1
    │
    ├─ IF spentMonthlyCents >= budgetMonthlyCents:
    │   → Auto-pause agent
    │   → Publish "agent.budget_exceeded" event
    │   → Dashboard shows alert
    │
    ├─ IF spentMonthlyCents >= 80% of budget:
    │   → Publish "agent.budget_warning" event
    │   → Dashboard shows warning
    │
    ▼
Cost Dashboard (user view):
    Total this month: $1,243
    ├── Engineers: $892
    ├── CEO: $37
    ├── Marketing: $134
    ├── Compute: $18
    └── Remaining budget: $757
```

## Flow 5: Real-time Updates

```
Service mutation (e.g., task completed)
    │
    ├─ publishEvent({ companyId, type, payload })
    │
    ▼
Redis PUBLISH channel:company:{id}
    │
    ▼
API Server subscribes to Redis
    │
    ▼
WebSocket connection for that company
    │
    ▼
JSON pushed to browser: { type: "issue.updated", payload: {...} }
    │
    ▼
React Query invalidates relevant queries
    │
    ▼
Dashboard / task board updates automatically
```

## Flow 6: API Key Entry (Non-Tech User)

```
User opens Company Settings → API Keys tab
    │
    ├─ Clicks "Add API Key"
    ├─ Selects provider: "Anthropic"
    ├─ Guided instructions with screenshots:
    │   "1. Go to console.anthropic.com"
    │   "2. Click API Keys → Create Key"
    │   "3. Copy and paste below"
    ├─ Pastes key: sk-ant-...
    │
    ▼
POST /api/companies/:id/api-keys
  { provider: "anthropic", key: "sk-ant-..." }
    │
    ▼
API Key Vault Service
    ├─ Validate key (test API call to Anthropic)
    ├─ IF invalid → return error with guidance
    ├─ Encrypt with AES-256
    ├─ Store in companyApiKeys table
    ├─ Hash for dedup (never store raw)
    │
    ▼
UI shows: ✅ Anthropic key active
    Key: sk-ant-...XXXX (masked)
    Last used: Never
    Status: Valid
```
