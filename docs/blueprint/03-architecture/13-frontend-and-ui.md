# 13 — Frontend & UI

## Framework & Stack

| Technology | Purpose |
|-----------|---------|
| React 19 | UI library |
| Vite | Build tool + dev server |
| React Router | Client-side routing |
| React Query | Server state management |
| Tailwind CSS 4 | Utility-first styling |
| shadcn/ui | Component library (Radix UI) |
| Lucide React | Icon library |

## Folder Structure

```
apps/web/src/
├── main.tsx
├── App.tsx
├── api/                    # API client layer
│   ├── client.ts           # Base fetch wrapper (base URL, credentials, error class)
│   ├── agents.ts
│   ├── companies.ts
│   ├── issues.ts
│   ├── approvals.ts
│   ├── costs.ts
│   ├── goals.ts
│   ├── projects.ts
│   ├── activity.ts
│   ├── dashboard.ts
│   ├── templates.ts
│   └── auth.ts
├── pages/                  # Route pages (one file per route)
│   ├── Dashboard.tsx
│   ├── Agents.tsx
│   ├── AgentDetail.tsx
│   ├── Issues.tsx
│   ├── IssueDetail.tsx
│   ├── Approvals.tsx
│   ├── Costs.tsx
│   ├── Org.tsx
│   ├── Settings.tsx
│   ├── Onboarding.tsx
│   └── Auth.tsx
├── components/
│   ├── ui/                 # shadcn/ui primitives (Button, Badge, Card, Dialog, etc.)
│   ├── layout/             # Layout, Sidebar, Breadcrumbs, TopBar
│   ├── agents/             # AgentCard, AgentStatusBadge, RunsList, TranscriptViewer
│   ├── issues/             # IssueCard, KanbanBoard, IssueDetailPanel, CommentThread
│   ├── dashboard/          # MetricCard, ActiveAgentsPanel, CostChart, ActivityFeed
│   └── onboarding/         # TemplateGallery, GoalInput, ApiKeySetup, TeamReview, LaunchStep
├── context/                # React Context providers
│   ├── AuthContext.tsx
│   ├── CompanyContext.tsx
│   ├── ThemeContext.tsx
│   └── NotificationContext.tsx
├── hooks/                  # Custom hooks
│   ├── useAgents.ts
│   ├── useIssues.ts
│   ├── useRealtime.ts
│   └── useCompany.ts
├── lib/                    # Utilities
│   ├── utils.ts            # cn(), formatters, date helpers
│   └── constants.ts
└── adapters/               # Per-adapter UI config fields
    ├── claude.tsx
    ├── openai-codex.tsx
    └── index.ts
```

## Design Philosophy: Non-Tech First

Every UI decision optimizes for users who have never used a terminal:

| Standard Dev Tool | Our Approach |
|------------------|-------------|
| JSON config editor | Visual form with dropdowns |
| Raw log viewer | Summary cards with "View Details" expand |
| Terminal output | Progress bars + status badges |
| API key field | Guided setup wizard with screenshots |
| "Create agent" form | "Hire from template" gallery |
| Git repo URL input | "Connect GitHub" OAuth button |

## Progressive Disclosure Pattern

```
Simple View (default for non-tech):
  ┌──────────────────────────────────┐
  │  ✅ Agent "Alice" completed task  │
  │  "Built login page"              │
  │  [View Summary]                  │
  └──────────────────────────────────┘

Advanced View (toggle for power users):
  ┌──────────────────────────────────┐
  │  Run #abc123 — succeeded         │
  │  Exit code: 0                    │
  │  Tokens: 85K in / 12K out       │
  │  Cost: $0.44                     │
  │  Duration: 4m 32s                │
  │  [View Full Transcript]          │
  │  [View Raw Logs]                 │
  └──────────────────────────────────┘
```

## Routing Architecture

### Public Routes (no auth)
```
/auth                    → Login / signup
/invite/:token           → Accept invite
/onboarding              → First-run setup wizard
```

### Dashboard Routes (authenticated)
```
/:company/dashboard      → Main metrics dashboard
/:company/team           → Agent list (simple view)
/:company/team/:id       → Agent detail + runs
/:company/tasks          → Task board
/:company/tasks/:id      → Task detail + comments
/:company/org            → Org chart visualization
/:company/approvals      → Pending approvals
/:company/costs          → Spending dashboard
/:company/activity       → Activity log
/:company/settings       → Company settings + API keys
/settings                → Account settings + billing
```

## Main Pages

### Onboarding Wizard (Critical for Non-Tech)
```
Step 1: "What kind of company?"
  → Template gallery (SaaS Startup, Content Agency, Dev Shop, etc.)

Step 2: "What's your goal?"
  → Free-text input: "Build a task management app"

Step 3: "Connect your AI provider"
  → Step-by-step API key setup with screenshots
  → "Get your Anthropic API key" → link + instructions
  → Paste key → validate → store encrypted

Step 4: "Review your team"
  → Pre-configured agents from template
  → User can adjust: add/remove agents, change names
  → Budget recommendation shown

Step 5: "Launch!"
  → One click → company created → agents start working
  → Redirect to dashboard
```

### Dashboard
- **Metric cards:** Active agents, tasks completed today, total cost this month, success rate
- **Active agents panel:** Who's working right now, on what
- **Recent activity:** Timeline of agent actions (summaries, not raw logs)
- **Cost chart:** Daily spending trend
- **Alerts:** Budget warnings, failed tasks, approval requests

### Team (Agent List)
- **Card grid** (not table) — agent avatar, name, role, status badge
- **Status filter:** All / Working / Idle / Paused / Error
- **Quick actions:** Pause, resume, view detail
- **"Hire Agent" button** → template picker or custom setup

### Agent Detail
- **Overview tab:** Role, status, cost this month, last activity
- **Activity tab:** Recent tasks worked on (summaries)
- **Runs tab:** Heartbeat history with expandable transcripts (advanced)
- **Settings tab:** Model, budget, permissions (advanced)

### Task Board
- **Kanban view:** Backlog → Todo → In Progress → In Review → Done
- **Each card:** Title, assignee avatar, priority badge, last update
- **Create task:** Simple title + description (AI can auto-enrich)
- **Task detail:** Properties panel, comment thread, attached files, live progress

### Org Chart
- **Tree visualization:** CEO at top, reporting lines flowing down
- **Each node:** Agent avatar, name, role, status dot
- **Click to expand:** Quick stats, recent activity

### Approvals
- **Notification-style list:** Agent requests with context
- **One-click actions:** Approve / Reject / Ask for Revision
- **Types:** Hire request, strategy proposal, budget increase

### Cost Dashboard
- **Total spend:** This month, trend vs last month
- **Per-agent breakdown:** Bar chart by agent
- **Per-task breakdown:** Highest cost tasks
- **Token usage:** Input vs output tokens
- **Alerts config:** Set threshold for notifications

### API Key Management (in Settings)
- **Add key:** Select provider (Anthropic, OpenAI, Google) → paste key → validate
- **Key status:** Active / Invalid / Expired
- **Usage:** Tokens consumed via this key
- **Security:** Keys encrypted at rest, never shown after entry

## State Management

### React Query (server state)
- 30-second stale time
- Refetch on window focus
- Mutation success → invalidate related queries
- Heartbeat polling: 10-15 second intervals

### React Context (client state)
| Context | Purpose |
|---------|---------|
| `CompanyContext` | Selected company, list of companies |
| `AuthContext` | User session, permissions |
| `ThemeContext` | Dark/light mode |
| `NotificationContext` | Toast messages, alert badges |

## API Client

`api/client.ts` — centralized fetch wrapper:
- Base URL: `/api`
- Cookie credentials for session auth
- Custom error class with `status` + `body`
- Resource modules (`agents.ts`, `issues.ts`, etc.) export typed functions

```typescript
// api/client.ts (excerpt)
export async function apiFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`/api${path}`, {
    ...init,
    credentials: 'include',
    headers: { 'Content-Type': 'application/json', ...init?.headers },
  });
  if (!res.ok) throw new ApiError(res.status, await res.json());
  return res.json();
}
```

## Real-time Updates

- WebSocket connection per company (via `RealtimeGateway` on the API)
- Events: agent status changes, task updates, run completions, cost alerts
- Fallback: polling at 10-15 second intervals
- Toast notifications for important events (approval request, task completed, budget alert)
- `useRealtime` hook manages connection lifecycle and dispatches events to React Query cache

## Adapter UI Config Fields

Each agent adapter type (claude, openai-codex, etc.) requires different configuration fields. The `adapters/` directory exports a React component per adapter that renders the appropriate form fields. These are used in the agent creation and settings forms.
