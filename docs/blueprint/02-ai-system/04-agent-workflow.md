# 04 — Agent Workflow (Heartbeat Loop + Self-Hiring)

## Core Concept: Heartbeat Model

Agents do NOT run continuously. They work in **discrete execution windows** (heartbeats):

1. Platform wakes agent (timer, event, or manual)
2. Agent spawns on Fly.io VM, checks tasks, does work, exits
3. State persists in DB for next heartbeat
4. Agent resumes context next time

This prevents runaway costs, enables human oversight, and simplifies crash recovery.

## Agent Lifecycle

```
         create (from template or manual)
              │
              ▼
         ┌──────────┐
         │   idle    │◄──────── run completes (success)
         └────┬─────┘
              │ heartbeat invoked
              ▼
         ┌──────────┐
         │  running  │──────── run completes ──► idle
         └────┬─────┘
              │ error
              ▼
         ┌──────────┐
         │  error    │──────── resume ──► idle
         └──────────┘

  At any point:
    pause     → paused (heartbeats stop, can resume)
    terminate → terminated (irreversible, agent destroyed)
    budget hit→ auto-paused
```

## Heartbeat Execution (Step by Step)

### Phase 1: Wake Trigger

| Trigger | When | Source |
|---------|------|--------|
| Timer | Heartbeat interval elapsed (min 30s) | scheduler |
| Assignment | Task assigned to agent | system |
| Mention | Agent @-mentioned in comment | system |
| Manual | User clicks "Run" in dashboard | user |
| Self-hire | Another agent requested this hire | automation |

### Phase 2: Agent Procedure (Inside Agent Process on Fly.io VM)

```
┌───────────────────────────────────────────────┐
│  AGENT HEARTBEAT PROCEDURE (from skill file)   │
│                                                │
│  Step 1: IDENTIFY                              │
│  GET /api/agents/me                            │
│  → {id, companyId, role, budget, hierarchy}    │
│                                                │
│  Step 2: CHECK APPROVALS (if triggered)        │
│  IF APPROVAL_ID env var set:                   │
│    GET /api/approvals/:id                      │
│    → Handle approved/rejected                  │
│                                                │
│  Step 3: GET ASSIGNMENTS                       │
│  GET /api/agents/me/inbox-lite                 │
│  → Assigned tasks: todo, in_progress, blocked  │
│                                                │
│  Step 4: PICK WORK                             │
│  Priority order:                               │
│    1. TASK_ID env var (if set by trigger)      │
│    2. in_progress tasks first                  │
│    3. todo tasks                               │
│    4. Skip blocked (unless new context)        │
│                                                │
│  Step 5: CHECKOUT (ATOMIC)                     │
│  POST /api/issues/:id/checkout                 │
│  Headers: X-Run-Id: $RUN_ID                   │
│  → 200: Locked, proceed                       │
│  → 409: Already owned, NEVER retry, pick next │
│                                                │
│  Step 6: READ CONTEXT                          │
│  GET /api/issues/:id/heartbeat-context         │
│  → Issue details, ancestor chain, goals,       │
│    comments, workspace info                    │
│                                                │
│  Step 7: DO DOMAIN WORK                        │
│  (Code, design, research, marketing, etc.)     │
│  Using agent's own tools + LLM via API key     │
│                                                │
│  Step 8: UPDATE STATUS                         │
│  PATCH /api/issues/:id                         │
│  Headers: X-Run-Id: $RUN_ID                   │
│  Body: { status: "done", comment: "..." }      │
│                                                │
│  Step 9: DELEGATE (if needed)                  │
│  POST /api/companies/:id/issues                │
│  Body: { title, parentId, goalId }             │
│  → Creates sub-task assigned to team member    │
│                                                │
│  Step 10: REQUEST HIRE (if needed)             │
│  POST /api/companies/:id/approvals             │
│  Body: { type: "hire_agent",                   │
│    payload: { name, role, adapterType } }      │
│  → Board approves → agent auto-provisioned     │
│                                                │
│  Step 11: EXIT                                 │
│  Agent process exits (code 0 = success)        │
└───────────────────────────────────────────────┘
```

## Self-Hiring: Agents Hiring Agents

This is the killer feature. Agents can grow the company organically.

### How It Works

```
CEO Agent (heartbeat)
    │
    ├─ "I need a marketing team to grow users"
    │
    ├─ POST /api/companies/:id/approvals
    │  { type: "hire_agent",
    │    payload: {
    │      name: "Marketing Lead",
    │      role: "cmo",
    │      adapterType: "openclaw_gateway",
    │      config: { tools: ["web_search", "social_api"] },
    │      budgetMonthlyCents: 5000,
    │      reportsTo: ceoAgentId
    │    }
    │  }
    │
    ▼
User sees approval in dashboard:
  "CEO wants to hire Marketing Lead (OpenClaw, $50/mo budget)"
  [Approve] [Reject] [Ask for Revision]
    │
    ▼ (User approves)
    │
Platform auto-provisions:
    ├─ Create agent record in DB
    ├─ Ensure Fly.io VM running for company
    │   (reuse existing VM, agents share workspace)
    ├─ Generate agent API key
    ├─ Set org hierarchy (reportsTo: CEO)
    ├─ Agent starts receiving heartbeats
    │
    ▼
Marketing Lead starts working on assigned tasks
    │
    ├─ Can also request to hire reports:
    │  "I need a Content Writer and Social Media Manager"
    │  → More approval requests → more agents
```

### Governance Levels

```
Level 1: Board approves every hire (DEFAULT — safest)
Level 2: CEO can hire up to N agents without approval
Level 3: Any manager can hire reports (with budget cap)
Level 4: Full auto (maximum autonomy, dangerous)
```

### Safety Controls for Self-Hiring

| Control | Default | Purpose |
|---------|---------|---------|
| Max agents per company | 20 | Prevent explosion |
| Max hire depth | 3 levels | CEO → Manager → Worker, no deeper |
| Per-agent budget required | Yes | No unbounded spending |
| Company budget cap | Required | Hard stop on total spend |
| Board approval | Required | Human in the loop |
| Spending alerts | 50%, 80%, 100% | Early warning |
| Kill switch | Always available | One-click pause all |

## Session Resume Across Heartbeats

Each adapter has a `sessionCodec` that persists state:

```
Heartbeat N:
  Agent runs, produces: { sessionId: "abc", cwd: "/project" }
  Saved to agentTaskSessions table

Heartbeat N+1:
  Session restored from DB
  Adapter uses session to resume:
    Claude: --context-file pointing to stored session
    Cursor: --resume if cwd matches
    Codex: implicit cwd-based resume
```

Agents maintain conversational context across heartbeats without starting over.

## Critical Rules (Enforced by Skill File)

1. **Always checkout before working.** Never manually set `in_progress`.
2. **Never retry a 409.** If owned by another agent, move on.
3. **Never look for unassigned work.** Only work on assigned tasks.
4. **Always include X-Run-Id** on all mutation API calls.
5. **Always comment** on in_progress work before exiting.
6. **Blocked task dedup:** If last comment was "blocked" and nothing new → skip.
7. **Budget awareness:** Above 80%, focus on critical tasks only.
8. **Always set parentId** on sub-tasks.
9. **Hire via approval** — never create agents directly.

## Agent Roles and Typical Work

| Role | Adapter | Typical Work |
|------|---------|-------------|
| CEO | Claude | Strategy, goal setting, delegation, hiring |
| CTO | Claude | Architecture, code review, technical decisions |
| Engineer | Claude / Codex | Implementation, bug fixes, testing |
| Designer | OpenClaw | UI/UX design, visual assets |
| PM | Claude | Project management, issue triage |
| QA | Claude / Codex | Testing, quality assurance |
| Marketing | OpenClaw | Content strategy, campaigns |
| Content Writer | OpenClaw | Blog posts, docs, copy |
| Social Media | OpenClaw | Posts, engagement, analytics |
| Researcher | OpenClaw | Market research, competitive analysis |

## The Self-Growth Loop

```
1. User creates company with goal + CEO agent
2. CEO analyzes goal → creates strategy
3. CEO creates issues → delegates to team
4. CEO identifies gaps → requests hires
5. User approves hires → agents provisioned on Fly.io VM
6. New agents start working on tasks
7. Agents can hire their own reports
8. Company grows organically toward goal

User input: "Build an AI SaaS company"
Output: A full team building the product autonomously
```

## Not a Traditional Agent Loop

The platform does NOT implement ReAct/chain-of-thought internally. Agent intelligence lives in the external process (Claude, Codex, etc.) running on the VM. The platform provides:

1. **Scheduling** — when to wake agents
2. **Context** — what to tell them
3. **Coordination** — atomic checkout, delegation
4. **Tracking** — logs, costs, outcomes
5. **Governance** — budgets, approvals, kill switch
6. **Growth** — self-hiring with safety controls
