# 03 — AI Workflow (Full Lifecycle)

This is the most critical document. Traces the complete lifecycle of an agent execution.

## End-to-End Flow

```
TRIGGER
  ├── User clicks "Run" on agent
  ├── Heartbeat timer interval elapsed
  ├── Task assigned to agent
  └── Agent mentioned in comment
       │
       ▼
┌─────────────────────────────────────┐
│  1. WAKEUP REQUEST                   │
│  POST /api/agents/:id/wakeup        │
│  → agentWakeupRequests record        │
│  → status: "queued"                  │
│  → Includes: taskId, commentId,      │
│    approvalId, reason                │
│  → Coalescing: merge duplicate wakes │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  2. VALIDATE                         │
│  a. Agent active? (not paused/       │
│     terminated)                      │
│  b. Under budget?                    │
│  c. No concurrent run active?        │
│  d. Adapter exists?                  │
│  IF FAIL → skip wakeup              │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  3. RETRIEVE USER'S API KEY          │
│  companyApiKeys table                │
│  → Decrypt AES-256 in memory         │
│  → Get key for agent's provider      │
│    (e.g., ANTHROPIC_API_KEY)         │
│  IF NO KEY → fail with clear error   │
│    "Missing Anthropic API key.       │
│     Add one in Company Settings."    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  4. ENSURE VM RUNNING                │
│  Provisioner.ensureMachine()         │
│  → Boot VM if hibernated (~3s)       │
│  → Reuse if already running          │
│  → Wait for Agent Executor ready     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  5. BUILD EXECUTION CONTEXT          │
│                                      │
│  ExecutionRequest = {                │
│    runId, agentId, companyId,        │
│    adapter: { type, config, model }, │
│    apiKey: "<decrypted-key>",        │
│    workspace: { cwd, repo, branch }, │
│    context: {                        │
│      taskId, comments, goals,        │
│      wakeReason, commentId           │
│    },                                │
│    session: { restored from DB },    │
│    authToken: "<agent-jwt-48h>",     │
│    skills: [ "task-protocol" ]       │
│  }                                   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  6. DISPATCH TO EXECUTION ENGINE     │
│                                      │
│  a. POST to Agent Executor on VM     │
│  b. Agent Executor builds env vars:  │
│     AGENT_ID, COMPANY_ID, RUN_ID,   │
│     API_URL, API_KEY (jwt),          │
│     ANTHROPIC_API_KEY (user's),      │
│     TASK_ID, WAKE_REASON,            │
│     WORKSPACE_CWD, ...              │
│  c. Symlink skill files              │
│  d. Spawn agent process:             │
│     claude/codex/cursor/etc.         │
│  e. Pipe prompt to stdin             │
│  f. Capture stdout/stderr            │
│  g. Enforce timeout                  │
│  h. Stream results back via SSE      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  7. AGENT EXECUTES                   │
│  (Inside agent process on Fly.io VM) │
│                                      │
│  a. GET /api/agents/me               │
│     → Identity, role, budget         │
│                                      │
│  b. GET /api/agents/me/inbox-lite    │
│     → Assigned tasks                 │
│                                      │
│  c. POST /api/issues/:id/checkout    │
│     → Atomic claim (409 if owned)    │
│                                      │
│  d. GET /api/issues/:id/             │
│        heartbeat-context             │
│     → Full task context              │
│                                      │
│  e. DO DOMAIN WORK                   │
│     (code, design, research, etc.)   │
│                                      │
│  f. PATCH /api/issues/:id            │
│     → Update status: done/blocked    │
│                                      │
│  g. POST /api/issues/:id/comments    │
│     → Document what was done         │
│                                      │
│  h. (Optional) Create sub-tasks      │
│     → POST /api/companies/:id/issues │
│                                      │
│  i. EXIT (code 0 = success)          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  8. PARSE RESULT                     │
│                                      │
│  { exitCode, signal, timedOut,       │
│    usage: { inputTokens,             │
│             outputTokens, costUsd }, │
│    sessionParams: { sessionId, cwd },│
│    provider, model, summary }        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  9. PERSIST & TRACK                  │
│                                      │
│  a. heartbeatRuns → status,          │
│     usageJson, resultJson            │
│  b. agentTaskSessions → session      │
│     state for resume                 │
│  c. agentRuntimeState → cumulative   │
│     token counters                   │
│  d. costEvents → per-event cost      │
│  e. agent.spentMonthlyCents +=       │
│  f. IF over budget → auto-pause      │
│  g. activityLog → audit entry        │
│  h. Compute cost → track Fly.io      │
│     seconds used                     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  10. NOTIFY UI                       │
│                                      │
│  publishEvent("heartbeat.completed") │
│  → Redis pub/sub                     │
│  → WebSocket → browser               │
│  → Dashboard updates in real-time    │
│                                      │
│  IF VM IDLE:                         │
│    Start idle timer                  │
│    → Hibernate VM after 10 min       │
└─────────────────────────────────────┘
```

## Error Handling

```
Agent fails:
  ├── Non-zero exit code → status = "failed"
  ├── Timeout → SIGTERM → SIGKILL → status = "timed_out"
  ├── Missing API key → status = "failed", error = "No API key configured"
  ├── Invalid API key → status = "failed", error = "API key rejected by provider"
  ├── VM boot failure → status = "failed", error = "Could not start cloud VM"
  └── Budget exceeded → agent auto-paused, no more runs

Orphan detection (every 30s):
  Runs stuck "running" > 5 minutes → marked "failed"
```

## Cost Tracking

```
Agent runs, adapter reports:
  { inputTokens: N, outputTokens: N, costUsd: N.NN }

Server processes:
  1. Insert costEvents (provider, model, tokens, cents)
  2. Update agentRuntimeState totals
  3. Update agent.spentMonthlyCents
  4. Track Fly.io compute seconds → computeCostCents
  5. IF spend >= budget → auto-pause agent
  6. IF spend >= 80% budget → alert to dashboard
```

## Streaming During Execution

```
Agent stdout/stderr → onLog() callback
  → heartbeatRunEvents record (seq number)
  → Redis publish
  → WebSocket push to browser
  → UI updates transcript view in real-time

UI polls at 10-15 second intervals (or receives WebSocket push)
```

## What Makes This Different

| Standard AI App | This Platform |
|----------------|---------------|
| App calls LLM API | Platform spawns agent; agent calls its own LLM |
| Single model | 9+ adapter types, any provider |
| Continuous | Discrete heartbeats (cost-safe) |
| User manages keys in .env | User enters key in UI, encrypted vault handles rest |
| Runs on a single machine | Per-company cloud VMs on Fly.io |
| No governance | Budget caps, approvals, kill switch |
| Single agent | Multi-agent org with hierarchy |
