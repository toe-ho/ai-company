# Phase 17: Frontend Pages

## Context Links
- [13 - Frontend & UI](../../docs/blueprint/03-architecture/13-frontend-and-ui.md) (pages section)
- [18 - API Response Schemas](../../docs/blueprint/04-data-and-api/18-api-response-schemas.md)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 5h
- **Description:** All 11 pages + domain-specific components. Onboarding wizard is highest priority (first user experience).

## Key Insights
- Non-tech first: card grids over tables, summaries over raw data, progressive disclosure
- Onboarding wizard: 5 steps, must work flawlessly for first-time users
- Kanban board for tasks (drag-and-drop optional for V1)
- Dashboard aggregates metrics from multiple queries
- Agent detail has 4 tabs: overview, activity, runs, settings

## Architecture
```
apps/web/src/
├── pages/
│   ├── auth-page.tsx
│   ├── onboarding-page.tsx
│   ├── dashboard-page.tsx
│   ├── agents-page.tsx
│   ├── agent-detail-page.tsx
│   ├── issues-page.tsx
│   ├── issue-detail-page.tsx
│   ├── org-page.tsx
│   ├── approvals-page.tsx
│   ├── costs-page.tsx
│   └── settings-page.tsx
├── components/
│   ├── agents/
│   │   ├── agent-card.tsx
│   │   ├── agent-status-badge.tsx
│   │   ├── runs-list.tsx
│   │   └── transcript-viewer.tsx
│   ├── issues/
│   │   ├── issue-card.tsx
│   │   ├── kanban-board.tsx
│   │   ├── issue-detail-panel.tsx
│   │   └── comment-thread.tsx
│   ├── dashboard/
│   │   ├── metric-card.tsx
│   │   ├── active-agents-panel.tsx
│   │   ├── cost-chart.tsx
│   │   └── activity-feed.tsx
│   └── onboarding/
│       ├── template-gallery.tsx
│       ├── goal-input.tsx
│       ├── api-key-setup.tsx
│       ├── team-review.tsx
│       └── launch-step.tsx
└── adapters/                     # Per-adapter UI config fields
    ├── claude-config-fields.tsx
    ├── codex-config-fields.tsx
    └── index.ts
```

## Related Code Files

### Files to Create
~30 files (11 pages + ~20 components).

## Implementation Steps

1. **Auth Page** (`auth-page.tsx`)
   - Login/signup form (email + password)
   - Toggle between login and signup modes
   - OAuth buttons: "Continue with Google", "Continue with GitHub"
   - Redirect to /onboarding on first login, /:companyId/dashboard on subsequent
   - Use AuthContext for state management

2. **Onboarding Wizard** (`onboarding-page.tsx`) -- CRITICAL
   - 5-step wizard with progress indicator
   - State managed locally (useState per step, accumulated into final payload)

   **Step 1: Template Gallery** (`template-gallery.tsx`)
   - Grid of template cards (icon, name, description, agent count)
   - Fetch from `GET /api/templates`
   - Select one -> highlight -> next

   **Step 2: Goal Input** (`goal-input.tsx`)
   - Large text input: "What do you want your AI company to do?"
   - Placeholder: "Build a task management app"
   - Optional: Company name field (auto-suggest from goal)

   **Step 3: API Key Setup** (`api-key-setup.tsx`)
   - Provider selector (Anthropic, OpenAI, Google)
   - Step-by-step instructions with external link to get key
   - Paste field (masked input)
   - "Validate" button -> calls POST validate endpoint
   - Success/failure badge
   - This is the highest friction step -- make it clear and forgiving

   **Step 4: Team Review** (`team-review.tsx`)
   - Cards for each pre-configured agent from template
   - Show: name, role, adapter, estimated cost
   - Allow: rename, remove agent, add agent
   - Budget recommendation shown
   - "This looks good" button

   **Step 5: Launch** (`launch-step.tsx`)
   - Summary: company name, goal, team size, budget estimate
   - Big "Launch Your AI Company" button
   - POST /api/companies/from-template
   - Loading state with progress animation
   - On success: redirect to /:companyId/dashboard

3. **Dashboard** (`dashboard-page.tsx`)
   - Uses `GET /api/companies/:cid/dashboard/summary`
   - **MetricCard**: Reusable card with icon, label, value, trend
     - Active agents, tasks completed today, cost this month, success rate
   - **ActiveAgentsPanel**: Who's working now, on what task, status dot
   - **ActivityFeed**: Recent actions as timeline (agent avatar + summary + timestamp)
   - **CostChart**: Daily spending bar chart (use recharts or chart.js)
   - Layout: 4-metric row, then 2-column (agents + activity), then full-width chart

4. **Agents Page** (`agents-page.tsx`)
   - Card grid layout (not table)
   - **AgentCard**: Avatar, name, role, status badge, cost this month, last activity
   - **AgentStatusBadge**: Color-coded (green=active, yellow=idle, blue=running, red=error, gray=paused)
   - Status filter tabs: All / Working / Idle / Paused / Error
   - "Hire Agent" button -> opens template picker dialog
   - Search by name

5. **Agent Detail** (`agent-detail-page.tsx`)
   - Tabs: Overview, Activity, Runs, Settings
   - **Overview tab**: Role, status, cost this month, budget usage bar, last heartbeat, assigned tasks
   - **Activity tab**: Recent tasks worked on (summaries, not raw logs)
   - **Runs tab**: HeartbeatRun history list with expandable transcript
     - **RunsList**: Date, status badge, duration, tokens, cost
     - **TranscriptViewer**: Scrollable log output with syntax highlighting (progressive disclosure)
   - **Settings tab**: Model config, budget, permissions (for advanced users)

6. **Issues/Tasks Page** (`issues-page.tsx`)
   - **KanbanBoard**: 5 columns (Backlog, Todo, In Progress, In Review, Done)
   - **IssueCard**: Title, assignee avatar, priority badge, last update
   - Create task: Simple form (title + description), POST to API
   - Filter by assignee, priority, project
   - V1: No drag-and-drop; use status dropdown to move cards

7. **Issue Detail** (`issue-detail-page.tsx`)
   - Left panel: Title, description (markdown rendered), status dropdown, priority, assignee, project, labels
   - Right panel (or below): Comment thread
   - **CommentThread**: Chronological comments with author avatar, agent/user badge, markdown content
   - Add comment form at bottom
   - File attachments section (upload + list)
   - Live progress indicator if currently being worked on (running heartbeat)

8. **Org Chart** (`org-page.tsx`)
   - Tree visualization: CEO at top, reporting lines flowing down
   - Use simple CSS tree layout (flex + connectors) or lightweight lib
   - Each node: Agent avatar, name, role, status dot
   - Click node: show quick stats popover

9. **Approvals Page** (`approvals-page.tsx`)
   - Notification-style list
   - Each card: Type badge (hire/strategy), requesting agent, summary, created time
   - One-click actions: Approve / Reject / Ask for Revision
   - Expandable details section

10. **Costs Page** (`costs-page.tsx`)
    - Total spend this month, trend vs last month
    - Per-agent bar chart
    - Per-task top-cost list
    - Token usage breakdown (input vs output)
    - Budget alert configuration

11. **Settings Page** (`settings-page.tsx`)
    - Company settings: name, description, brand color
    - API key management: list stored keys (masked), add new, validate, delete
    - Team management: invite users (future)
    - Billing info (future placeholder)

12. **Adapter config field components** (`adapters/`)
    - `claude-config-fields.tsx`: Model selector, effort level, timeout, dangerously skip permissions toggle
    - `codex-config-fields.tsx`: Model selector, search toggle
    - `index.ts`: Registry mapping adapter type to component

## Todo List
- [ ] Auth page (login/signup + OAuth)
- [ ] Onboarding wizard (5 steps + 5 step components)
- [ ] Dashboard (4 metric cards + 3 panels)
- [ ] Agents page (card grid + status filter)
- [ ] Agent detail (4 tabs)
- [ ] Issues page (Kanban board)
- [ ] Issue detail (properties + comments)
- [ ] Org chart (tree visualization)
- [ ] Approvals page
- [ ] Costs page
- [ ] Settings page
- [ ] Adapter config field components
- [ ] typecheck passes
- [ ] All pages render without errors

## Success Criteria
- Onboarding wizard completes full flow: template -> goal -> API key -> team -> launch
- Dashboard displays all metric areas
- Kanban board shows tasks in correct columns
- Agent detail shows runs with expandable transcripts
- All pages responsive (mobile-friendly)
- `turbo typecheck` passes

## Risk Assessment
- **Onboarding API key step:** Highest friction point; must have clear instructions and graceful error handling
- **Chart library:** Keep simple (recharts); don't over-invest in visualizations for V1
- **Kanban drag-and-drop:** Skip for V1; use dropdown to change status

## Security Considerations
- API keys masked after entry (show only last 4 chars)
- No sensitive data in URL params
- CSRF protection via cookie auth (SameSite)

## Next Steps
- Phase 18: Testing covers critical user flows
