# Design Guidelines

UI/UX design principles and implementation standards for AI Company Platform v2.

---

## Design Philosophy

### Non-Technical First
**Users are entrepreneurs with no coding background.** Avoid jargon. Use business concepts (CEO, tasks, budgets) instead of technical terms.

✅ **Good:**
- "Your company has completed 3 tasks this week"
- "Budget remaining: $500 of $1000"
- "Engineer is currently working"

❌ **Bad:**
- "3 heartbeat_runs completed"
- "Remaining tokens: 50K"
- "Agent status: EXECUTING"

### Progressive Disclosure
Show beginner defaults first. Advanced options hidden until needed.

```
Beginner view:
├─ Dashboard (simple metrics)
├─ Team (list of agents)
└─ Tasks (kanban board)

Advanced view (Settings → Developer):
├─ Agent adapter config
├─ Cost breakdowns by tool
├─ Execution logs
└─ API endpoints
```

### Real-time Transparency
Users watch agents work in real-time. Builds trust.

```
Execution view:
┌─ Task: "Write homepage"
├─ Status: In Progress (40%)
├─ Live log:
│  └─ "Writing index.tsx..."
│  └─ "Installing dependencies..."
│  └─ "Running tests..."
└─ Cost so far: $0.08
```

### Fail-Safe Design
Users retain override authority. Cost kill-switch. Approval workflows for risky decisions.

---

## Component Library

**Stack:** Tailwind CSS 4 + shadcn/ui + React 19

### shadcn/ui Components Used

| Component | Use Case | Example |
|-----------|----------|---------|
| `Button` | Actions | "Create Agent", "Save", "Execute" |
| `Card` | Content containers | Agent profile, task summary |
| `Dialog` | Forms & confirmations | Create company, approve hire |
| `Input` | Text entry | Name, goal, API key |
| `Select` | Dropdown choices | Agent role, adapter type |
| `Table` | Data display | Agent list, cost breakdown |
| `Tabs` | Section navigation | Dashboard tabs (overview, agents, tasks) |
| `Badge` | Status labels | "Active", "Paused", "Error" |
| `Progress` | Progress bars | Task completion, budget usage |
| `Toast` | Notifications | "Task created", "Error occurred" |
| `Tooltip` | Help text | Hover hints on complex fields |
| `Dialog + Form` | Multi-step wizards | Onboarding, agent config |

### Tailwind CSS Utilities

**Color palette:**
```
Primary: blue-600 (actions, highlights)
Success: green-600 (completed, healthy)
Warning: amber-600 (budget alerts, warnings)
Error: red-600 (failures, alerts)
Neutral: gray-100–900 (backgrounds, text)
```

**Example layout:**
```tsx
<div className="flex flex-col gap-4 p-6 bg-white rounded-lg border border-gray-200">
  <h2 className="text-lg font-bold text-gray-900">Tasks</h2>

  <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
    {tasks.map((task) => (
      <Card key={task.id} className="p-4">
        <div className="flex justify-between items-start">
          <div>
            <h3 className="font-semibold">{task.title}</h3>
            <p className="text-sm text-gray-600">{task.description}</p>
          </div>
          <Badge variant={task.status === 'done' ? 'default' : 'secondary'}>
            {task.status}
          </Badge>
        </div>
      </Card>
    ))}
  </div>
</div>
```

---

## Page Layouts

### 1. Authentication Pages (Login, Signup, Forgot Password)

**Layout:** Centered, minimal, dark background

```
┌─────────────────────────────────┐
│                                 │
│                                 │
│      ┌─────────────────────┐    │
│      │   AI Company        │    │
│      │   [logo]            │    │
│      │                     │    │
│      │ Email: [_______]    │    │
│      │ Pass:  [_______]    │    │
│      │                     │    │
│      │ [Login]             │    │
│      │ Sign up? →          │    │
│      └─────────────────────┘    │
│                                 │
│                                 │
└─────────────────────────────────┘
```

**Components:** Input, Button, Link, Toast (errors)

### 2. Onboarding Wizard

**Layout:** Multi-step form, progress indicator

```
Step 1/4: Company Details
┌──────────────────────────────────────┐
│ ◐ 1 2 3 4                            │
├──────────────────────────────────────┤
│ Company Name:  [________________]    │
│ Your Goal:     [________________]    │
│ Initial Budget: $[___]               │
├──────────────────────────────────────┤
│ [Cancel]                  [Next →]   │
└──────────────────────────────────────┘
```

**Steps:**
1. Company setup (name, goal, budget)
2. API keys (Anthropic, OpenAI, etc.)
3. Initial team (CEO, engineers, designers)
4. Review & confirm

**Components:** Card, Stepper, Input, Select, Button

### 3. Dashboard

**Layout:** 3-column: sidebar + main + right panel

```
┌─────────┬─────────────────────────────┬──────────┐
│ Logo    │ Dashboard                   │ User ▼   │
│ NAV     │                             │          │
├─────────┼─────────────────────────────┼──────────┤
│ • Home  │ ┌─ Quick Metrics ─────────┐ │ ┌─ Live  │
│ • Team  │ │ Tasks: 10                │ │ │ Status │
│ • Tasks │ │ Budget: $850 / $1000     │ │ │ ──── │
│ • Costs │ │ Agents: 5 / 5 active     │ │ │ CEO:  │
│ • Settg │ └──────────────────────────┘ │ │ ✓ Busy │
│         │                              │ │ ──── │
│         │ ┌─ Recent Activity ────────┐ │ │ Eng1: │
│         │ │ Agent-CEO completed task │ │ │ ✓ Idle│
│         │ │ Agent-Eng assigned issue │ │ │ ──── │
│         │ └──────────────────────────┘ │ └──────┘
│         │                              │          │
└─────────┴─────────────────────────────┴──────────┘
```

**Components:** Sidebar, Navbar, Card, Tabs, Badge, Chart

### 4. Team Management

**Layout:** Table + detail panel

```
┌─────────┬───────────────────────────────────────┐
│ NAV     │ Agents                                │
├─────────┼───────────────────────────────────────┤
│         │ [Add Agent] [Org Chart] [Settings] │
│         │                                       │
│         │ ┌─────────────────────────────────┐  │
│         │ │ Name      Role      Budget Status│  │
│         │ ├─────────────────────────────────┤  │
│         │ │ CEO       CEO       $100  Active │  │
│         │ │ Alice     Engineer  $80   Active │  │
│         │ │ Bob       Designer  $60   Active │  │
│         │ │ Carol     Marketer  $50   Paused │  │
│         │ └─────────────────────────────────┘  │
│         │                                       │
└─────────┴───────────────────────────────────────┘
```

**Components:** Table, Button, Badge, Dialog (add/edit), Select

### 5. Task Board (Kanban)

**Layout:** Horizontal columns (backlog, todo, in_progress, done)

```
┌─────────┬─────────────────────────────────────┐
│ NAV     │ Tasks                               │
├─────────┼─────────────────────────────────────┤
│         │ [Filter] [Assign] [+New]            │
│         │                                     │
│         │ Backlog      Todo        In Progress│
│         │ ┌────────┐ ┌────────┐ ┌───────────┐│
│         │ │Write   │ │Design  │ │Fix bug    ││
│         │ │API     │ │UI      │ │(60% done) ││
│         │ │        │ │        │ │           ││
│         │ │[Assign]│ │[Assign]│ │[View]     ││
│         │ └────────┘ └────────┘ └───────────┘│
│         │                                     │
│         │                       Done          │
│         │                       ┌───────────┐ │
│         │                       │Deploy app │ │
│         │                       │✓ Complete │ │
│         │                       └───────────┘ │
│         │                                     │
└─────────┴─────────────────────────────────────┘
```

**Components:** Card, Badge, Drag-drop (optional), Button

### 6. Task Detail / Execution Log

**Layout:** Vertical split: left (metadata) + right (log stream)

```
┌─────────┬─────────────────────┬───────────────┐
│ NAV     │ Task: "Write API"   │ Live Execution│
├─────────┼─────────────────────┼───────────────┤
│         │ Status: In Progress │ [13:45] Start │
│         │ Assigned: Engineer1 │ [13:46] Write │
│         │ Priority: High      │ [13:47] Test  │
│         │ Due: Today          │ [13:48] Done  │
│         │                     │               │
│         │ Description:        │ Cost: $0.12   │
│         │ Implement API route │ Tokens: 4700  │
│         │ for user auth       │               │
│         │                     │ [Save] [Close]│
│         │                     │               │
│         │ [Edit] [Delete]     │               │
│         │                     │               │
│         │ Comments:           │               │
│         │ ┌────────────────┐  │               │
│         │ │Your comment... │  │               │
│         │ │[Comment] [±]   │  │               │
│         │ └────────────────┘  │               │
│         │                     │               │
└─────────┴─────────────────────┴───────────────┘
```

**Components:** Card, Textarea, Badge, Log viewer, Buttons

### 7. Approval Workflows

**Layout:** List of pending decisions with accept/reject

```
┌─────────┬───────────────────────────────────┐
│ NAV     │ Approvals (3 pending)             │
├─────────┼───────────────────────────────────┤
│         │ 1. Agent: Engineer1 wants to hire │
│         │    Role: Junior Engineer, Cost: $5K│
│         │    [Approve]    [Reject]          │
│         │                                   │
│         │ 2. Budget Alert: CEO exceeded $800│
│         │    Current spend: $850 of $1000   │
│         │    [Pause Agent]  [Increase]      │
│         │                                   │
│         │ 3. Cost Override: Designer needs  │
│         │    +$100 for tool subscriptions   │
│         │    [Approve]    [Reject]          │
│         │                                   │
└─────────┴───────────────────────────────────┘
```

**Components:** Card, Button, Badge, Confirmation Dialog

### 8. Cost Dashboard

**Layout:** Metrics + charts + detailed breakdown

```
┌─────────┬──────────────────────────────────┐
│ NAV     │ Billing & Costs                  │
├─────────┼──────────────────────────────────┤
│         │ This Month:      $320 / $1000    │
│         │ Daily Average:   $10.67          │
│         │ Forecast (30d):  $640 / $1000    │
│         │                                  │
│         │ ┌─ By Agent ────────────────────┐│
│         │ │ CEO: $150 (47%)         [██ ]││
│         │ │ Eng: $100 (31%)         [██ ]││
│         │ │ Des: $50  (16%)         [█  ]││
│         │ │ Mkr: $20  (6%)          [    ]││
│         │ └────────────────────────────────┘│
│         │                                  │
│         │ ┌─ By Tool ─────────────────────┐│
│         │ │ Claude API: $280 (88%)  [██ ]││
│         │ │ Compute: $40 (12%)      [█  ]││
│         │ └────────────────────────────────┘│
│         │                                  │
└─────────┴──────────────────────────────────┘
```

**Components:** Card, Chart (recharts or Chart.js), Badge, Progress

### 9. Settings

**Layout:** Tabs for different settings sections

```
┌─────────┬──────────────────────────────────┐
│ NAV     │ Settings                         │
├─────────┼──────────────────────────────────┤
│         │ [Profile] [API Keys] [Billing]   │
│         │                                  │
│         │ Profile:                         │
│         │ Name: [Your Name________]        │
│         │ Email: [you@example.com]         │
│         │ [Logout]                         │
│         │                                  │
│         │ Notifications:                   │
│         │ ☑ Budget alerts                  │
│         │ ☑ Task completion                │
│         │ ☐ Daily digest                   │
│         │                                  │
│         │ [Save Changes]                   │
│         │                                  │
└─────────┴──────────────────────────────────┘
```

**Components:** Tabs, Input, Checkbox, Select, Button

---

## Interaction Patterns

### Creating a New Agent

```
1. User clicks [Add Agent]
   ↓
2. Dialog opens:
   Role: [Select ▼]
        - CEO
        - Engineer
        - Designer
        - Marketer
   Budget: $[___] / $[total]
   Capabilities: [Editor, Code Read/Write, Tool Use]
   ↓
3. User clicks [Create]
   ↓
4. Success toast: "Engineer-2 created. Ready to assign tasks."
   ↓
5. Dialog closes, agent appears in list
```

### Assigning a Task

```
1. User views task in backlog
   ↓
2. User clicks [Assign]
   ↓
3. Agent selector popover:
   [CEO] [Engineer-1] [Engineer-2] [Designer]
   ↓
4. User clicks engineer
   ↓
5. Task moves to "Todo" column
   ↓
6. Toast: "Assigned to Engineer-1"
   ↓
7. Live status updates: "Engineer-1 accepted task"
```

### Monitoring Execution

```
1. User views task detail
   ↓
2. Task status: "In Progress (40%)"
   ↓
3. Live log stream (auto-scrolls):
   [13:45] Agent woke up
   [13:46] Checked out task
   [13:47] Opened file editor
   [13:48] Writing code...
   ↓
4. Real-time cost: $0.08 so far
   ↓
5. Task completes:
   Status: "Done" (green badge)
   Final cost: $0.12
   ↓
6. Toast: "Task completed! Cost: $0.12"
```

### Emergency Cost Override

```
1. Budget alert triggered
   Agent exceeded monthly limit
   ↓
2. Agent pauses automatically (red badge)
   ↓
3. User sees notification:
   "CEO budget exceeded. Agent paused."
   ↓
4. User options:
   [Pause this agent]  [Increase budget]  [Review spending]
   ↓
5. If [Increase budget]:
   Dialog: "New limit: $[___]"
   Agent resumes immediately
   ↓
6. Toast: "Budget increased. Agent resumed."
```

---

## Responsive Design

### Breakpoints (Tailwind)

```
Mobile:   < 640px   (sm)  - Single column
Tablet:   640-1024px (md) - Two columns
Desktop:  > 1024px  (lg) - Full layout
```

### Mobile Layout

```
Mobile (stacked):
┌─────────────────────┐
│ ☰ Menu              │
├─────────────────────┤
│ Dashboard           │
├─────────────────────┤
│ ┌─────────────────┐ │
│ │ Quick Metrics   │ │
│ │ Tasks: 10       │ │
│ │ Budget: $850    │ │
│ └─────────────────┘ │
├─────────────────────┤
│ ┌─────────────────┐ │
│ │ Recent Activity │ │
│ │ - Task done     │ │
│ └─────────────────┘ │
└─────────────────────┘
```

**Example:**
```tsx
<div className="flex flex-col md:flex-row gap-4">
  {/* Mobile: stacked, Desktop: side-by-side */}
  <div className="w-full md:w-1/3">Left panel</div>
  <div className="w-full md:w-2/3">Main content</div>
</div>
```

---

## Dark / Light Mode

### Toggle Switch

```
Settings → Appearance → [Light] [Dark] [Auto]
```

**CSS Variables:**
```css
:root {
  --bg-primary: #ffffff;
  --text-primary: #000000;
  --border: #e5e7eb;
}

[data-theme="dark"] {
  --bg-primary: #1f2937;
  --text-primary: #ffffff;
  --border: #374151;
}
```

**Implementation:**
```tsx
export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark' | 'auto'>('auto');

  useEffect(() => {
    const actual = theme === 'auto' ? prefersDark ? 'dark' : 'light' : theme;
    document.documentElement.setAttribute('data-theme', actual);
  }, [theme]);

  return children;
}
```

---

## Accessibility (A11y)

### WCAG 2.1 AA Compliance

**Color contrast:** 4.5:1 for text

```css
/* ✅ Good contrast */
color: #000;
background: #fff;
/* Contrast: 21:1 */

/* ❌ Bad contrast */
color: #999;
background: #eee;
/* Contrast: 1.5:1 */
```

**Semantic HTML:**
```tsx
// ✅ Good: Semantic elements
<header><h1>Dashboard</h1></header>
<nav><a href="/tasks">Tasks</a></nav>
<main><Card /></main>
<footer>© 2026</footer>

// ❌ Bad: Divs everywhere
<div className="header"><div>Dashboard</div></div>
<div className="nav"><div>Tasks</div></div>
```

**Keyboard Navigation:**
```
Tab     → Move focus
Shift+Tab → Move back
Enter   → Activate button
Space   → Toggle checkbox
Esc     → Close dialog
```

**Aria Labels:**
```tsx
<button aria-label="Close menu" onClick={close}>×</button>
<div role="status" aria-live="polite">Task created</div>
<input aria-describedby="help-text" />
<span id="help-text">Budget in USD</span>
```

---

## Loading States

### Skeleton Loading

```tsx
// Before data loads
<div className="animate-pulse">
  <div className="h-4 bg-gray-300 rounded w-1/4 mb-2"></div>
  <div className="h-3 bg-gray-300 rounded w-full"></div>
</div>

// After data loads
<TaskCard task={task} />
```

### Progress Indicators

```
Linear (% completion):
Progress: [██████░░░░] 60%

Circular (waiting):
[⟳ Loading...]

Spinner (indeterminate):
[⊗ Processing...]
```

---

## Error Handling

### Toast Notifications

```tsx
// Success
toast.success('Task created');

// Error
toast.error('Failed to save. Try again.');

// Warning
toast.warning('Budget nearing limit');

// Info
toast.info('New features available');
```

### Form Validation

```tsx
<input
  type="number"
  placeholder="Budget ($)"
  min="100"
  max="100000"
  required
  aria-invalid={errors.budget ? 'true' : 'false'}
  aria-describedby={errors.budget ? 'budget-error' : undefined}
/>
{errors.budget && (
  <span id="budget-error" className="text-red-600">
    Budget must be between $100 and $100,000
  </span>
)}
```

### Error Boundaries

```tsx
<ErrorBoundary fallback={<ErrorPage />}>
  <DashboardPage />
</ErrorBoundary>
```

---

## Animation & Transitions

### Page Transitions

```tsx
<motion.div
  initial={{ opacity: 0 }}
  animate={{ opacity: 1 }}
  transition={{ duration: 0.3 }}
>
  <Dashboard />
</motion.div>
```

### Button Hover

```css
button {
  background: blue;
  transition: background 200ms ease;
}

button:hover {
  background: darkblue;
}
```

### Loading Animation

```css
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.loading { animation: pulse 2s infinite; }
```

---

## Typography

### Font Stack

```css
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
```

### Scales

| Element | Size | Weight | Line Height |
|---------|------|--------|-------------|
| H1 (Page title) | 28px | 700 | 1.2 |
| H2 (Section) | 22px | 600 | 1.3 |
| H3 (Subsection) | 18px | 600 | 1.4 |
| Body | 14px | 400 | 1.5 |
| Small (Help) | 12px | 400 | 1.4 |
| Mono (Code) | 13px | 400 | 1.6 |

---

## File Organization

```
src/components/
├── ui/                    # shadcn/ui (generated)
│   ├── button.tsx
│   ├── card.tsx
│   ├── dialog.tsx
│   └── ...
├── layout/                # Layout components
│   ├── Sidebar.tsx
│   ├── Header.tsx
│   └── Footer.tsx
├── forms/                 # Form-specific
│   ├── CompanyForm.tsx
│   ├── AgentConfigForm.tsx
│   └── IssueForm.tsx
├── charts/                # Visualizations
│   ├── SpendingChart.tsx
│   └── PerformanceChart.tsx
└── common/                # Reusable widgets
    ├── TaskCard.tsx
    ├── AgentBadge.tsx
    └── CostIndicator.tsx
```

---

**Version:** 1.0
**Last Updated:** 2026-03-17
**Framework:** React 19 + Tailwind CSS 4 + shadcn/ui
**See Also:** `./codebase-summary.md` (Frontend structure), `./code-standards.md` (React patterns)
