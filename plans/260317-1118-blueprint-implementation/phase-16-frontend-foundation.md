# Phase 16: Frontend Foundation

## Context Links
- [13 - Frontend & UI](../../docs/blueprint/03-architecture/13-frontend-and-ui.md)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 3h
- **Description:** Vite + React 19 setup with Tailwind CSS 4, shadcn/ui, React Router, React Query, API client, context providers, custom hooks, layout components.

## Key Insights
- Design philosophy: "non-tech first" -- visual forms over JSON, summaries over raw logs
- React Query for server state (30s stale time, refetch on focus)
- React Context for client state (auth, company, theme, notifications)
- API client is a thin fetch wrapper with `/api` base path + cookie credentials
- WebSocket connection per company for real-time events
- Tailwind CSS 4 (new engine, @import based)
- **Testing:** Vitest (not Jest) — Vite-native, faster, better ESM support
- **Routing:** React Router 7
<!-- Updated: Validation Session 1 - React Router 7 + Vitest -->

## Architecture
```
apps/web/src/
├── main.tsx
├── App.tsx                       # Router + providers wrapper
├── api/
│   ├── client.ts                 # Base fetch wrapper
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
├── components/
│   ├── ui/                       # shadcn/ui primitives (installed via CLI)
│   └── layout/
│       ├── app-layout.tsx
│       ├── sidebar.tsx
│       ├── top-bar.tsx
│       └── breadcrumbs.tsx
├── context/
│   ├── auth-context.tsx
│   ├── company-context.tsx
│   ├── theme-context.tsx
│   └── notification-context.tsx
├── hooks/
│   ├── use-agents.ts
│   ├── use-issues.ts
│   ├── use-realtime.ts
│   └── use-company.ts
├── lib/
│   ├── utils.ts                  # cn(), formatCents(), formatDate()
│   └── constants.ts
├── pages/                        # (Phase 17)
└── styles/
    └── globals.css               # Tailwind directives
```

## Related Code Files

### Files to Create
~30 files (API client, contexts, hooks, layout, config).

## Implementation Steps

1. **Setup Vite + React 19**
   - `apps/web/vite.config.ts`: React plugin, proxy `/api` to `http://localhost:3100`
   - `apps/web/index.html`: root div, script src main.tsx
   - `apps/web/src/main.tsx`: `createRoot(document.getElementById('root')!).render(<App />)`

2. **Setup Tailwind CSS 4**
   - Install `tailwindcss@4`, `@tailwindcss/vite`
   - `globals.css`:
     ```css
     @import "tailwindcss";
     ```
   - Add Vite plugin in vite.config.ts

3. **Setup shadcn/ui**
   - `npx shadcn@latest init` (select New York style, neutral color)
   - Install core primitives: Button, Card, Badge, Dialog, DropdownMenu, Input, Label, Select, Tabs, Toast, Tooltip, Sheet, Skeleton, Avatar, Separator
   - Components land in `components/ui/`

4. **Create API client** (`api/client.ts`)
   ```typescript
   export class ApiError extends Error {
     constructor(public status: number, public body: unknown) {
       super(`API Error ${status}`);
     }
   }

   export async function apiFetch<T>(path: string, init?: RequestInit): Promise<T> {
     const res = await fetch(`/api${path}`, {
       ...init,
       credentials: 'include',
       headers: { 'Content-Type': 'application/json', ...init?.headers },
     });
     if (!res.ok) throw new ApiError(res.status, await res.json().catch(() => null));
     return res.json();
   }
   ```

5. **Create resource API modules** (one per domain)
   ```typescript
   // api/agents.ts
   export const agentsApi = {
     list: (companyId: string) => apiFetch<Agent[]>(`/companies/${companyId}/agents`),
     get: (id: string) => apiFetch<Agent>(`/agents/${id}`),
     create: (companyId: string, data: CreateAgentDto) =>
       apiFetch<Agent>(`/companies/${companyId}/agents`, { method: 'POST', body: JSON.stringify(data) }),
     pause: (id: string) => apiFetch(`/agents/${id}/pause`, { method: 'POST' }),
     resume: (id: string) => apiFetch(`/agents/${id}/resume`, { method: 'POST' }),
     // ... etc
   };
   ```
   Repeat pattern for companies, issues, approvals, costs, goals, projects, activity, dashboard, templates, auth.

6. **Create React Query setup** (`App.tsx`)
   ```typescript
   const queryClient = new QueryClient({
     defaultOptions: {
       queries: { staleTime: 30_000, refetchOnWindowFocus: true },
     },
   });
   ```

7. **Create context providers**

   **AuthContext**: Manages user session, login/logout state
   ```typescript
   // Fetches /api/auth/get-session on mount
   // Provides: user, isAuthenticated, login(), logout()
   ```

   **CompanyContext**: Selected company, list of user's companies
   ```typescript
   // Provides: currentCompany, companies, selectCompany()
   // Persists selected company in localStorage
   ```

   **ThemeContext**: Dark/light mode toggle, persists in localStorage

   **NotificationContext**: Toast message queue, push/dismiss

8. **Create custom hooks**

   **useAgents(companyId)**:
   ```typescript
   export function useAgents(companyId: string) {
     return useQuery({
       queryKey: ['agents', companyId],
       queryFn: () => agentsApi.list(companyId),
     });
   }
   ```

   **useIssues(companyId, filters?)**: Same pattern with filters as query params

   **useCompany()**: Shorthand for CompanyContext consumer

   **useRealtime(companyId)**:
   - Connect WebSocket to `/api/ws`
   - Send `join` message with companyId
   - On events: invalidate relevant React Query cache
   - Cleanup: leave room + disconnect on unmount

9. **Create layout components**

   **AppLayout**: Sidebar + TopBar + main content area
   ```tsx
   export function AppLayout({ children }: { children: React.ReactNode }) {
     return (
       <div className="flex h-screen">
         <Sidebar />
         <div className="flex-1 flex flex-col">
           <TopBar />
           <main className="flex-1 overflow-auto p-6">{children}</main>
         </div>
       </div>
     );
   }
   ```

   **Sidebar**: Company name, nav links (Dashboard, Team, Tasks, Approvals, Costs, Settings), company switcher

   **TopBar**: Breadcrumbs, user avatar, notification bell

   **Breadcrumbs**: Auto-generated from current route

10. **Setup React Router** (`App.tsx`)
    ```typescript
    const router = createBrowserRouter([
      { path: '/auth', element: <AuthPage /> },
      { path: '/onboarding', element: <OnboardingPage /> },
      {
        path: '/:companyId',
        element: <AppLayout><Outlet /></AppLayout>,
        children: [
          { path: 'dashboard', element: <DashboardPage /> },
          { path: 'team', element: <AgentsPage /> },
          { path: 'team/:id', element: <AgentDetailPage /> },
          { path: 'tasks', element: <IssuesPage /> },
          { path: 'tasks/:id', element: <IssueDetailPage /> },
          { path: 'org', element: <OrgPage /> },
          { path: 'approvals', element: <ApprovalsPage /> },
          { path: 'costs', element: <CostsPage /> },
          { path: 'settings', element: <SettingsPage /> },
        ],
      },
    ]);
    ```

11. **Create `lib/utils.ts`**
    - `cn()`: Tailwind class merger (clsx + tailwind-merge)
    - `formatCents(cents: number)`: "$12.34"
    - `formatDate(date: string)`: Relative time ("2 hours ago")
    - `formatTokens(count: number)`: "85K"

## Todo List
- [ ] Vite + React 19 + Tailwind CSS 4 setup
- [ ] shadcn/ui init + core primitives
- [ ] API client (base + 11 resource modules)
- [ ] React Query setup
- [ ] 4 context providers
- [ ] 4 custom hooks
- [ ] Layout components (AppLayout, Sidebar, TopBar, Breadcrumbs)
- [ ] React Router structure
- [ ] Utility functions
- [ ] `turbo typecheck` passes
- [ ] `turbo dev --filter=@ai-company/web` starts successfully

## Success Criteria
- Dev server starts and renders layout
- API client proxy to backend works
- React Query fetches data with correct stale time
- WebSocket connects and receives events
- Tailwind + shadcn/ui components render correctly
- `turbo typecheck` passes

## Risk Assessment
- **Tailwind CSS 4 breaking changes:** New @import syntax; ensure Vite plugin compatible
- **shadcn/ui version:** Use latest compatible with React 19
- **WebSocket reconnection:** Implement reconnect logic in useRealtime hook

## Security Considerations
- API client uses credentials: 'include' for cookie auth
- No tokens stored in localStorage (session cookies only)
- XSS prevention: React escapes by default; avoid dangerouslySetInnerHTML

## Next Steps
- Phase 17: Page components use these foundations
