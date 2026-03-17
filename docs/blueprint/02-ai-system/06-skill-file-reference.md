# 06 — Skill File Reference (Complete Agent Instruction Document)

This is the **exact content** that goes into `skills/task-protocol/SKILL.md` — the master instruction document every agent receives. This file is the single most important piece of the system. Without it, agents don't know how to interact with the platform.

---

```markdown
---
name: task-protocol
description: >
  Interact with the platform control plane API to manage tasks, coordinate with
  other agents, and follow company governance. Use when you need to check
  assignments, update task status, delegate work, post comments, or call any
  platform API endpoint. Do NOT use for the actual domain work itself (writing
  code, research, etc.) — only for platform coordination.
---

# Task Protocol Skill

You run in **heartbeats** — short execution windows triggered by the platform. Each heartbeat, you wake up, check your work, do something useful, and exit. You do not run continuously.

## Authentication

Env vars auto-injected:

| Variable | Purpose |
|----------|---------|
| `AGENT_ID` | Your agent UUID |
| `COMPANY_ID` | Your company UUID |
| `API_URL` | Control plane API base URL |
| `RUN_ID` | Current heartbeat run UUID |
| `API_KEY` | Short-lived JWT for this run |

Optional wake-context vars (set when triggered by specific events):

| Variable | Purpose |
|----------|---------|
| `TASK_ID` | Issue/task that triggered this wake |
| `WAKE_REASON` | Why this run was triggered (e.g., `assignment`, `issue_comment_mentioned`) |
| `WAKE_COMMENT_ID` | Specific comment that triggered this wake |
| `APPROVAL_ID` | Approval that was resolved |
| `APPROVAL_STATUS` | Status of that approval (`approved`, `rejected`, `revision_requested`) |
| `LINKED_ISSUE_IDS` | Comma-separated issue IDs linked to approval |

All requests use `Authorization: Bearer $API_KEY`. All endpoints under `/api`, all JSON. Never hard-code the API URL.

**Run audit trail:** You MUST include `-H 'X-Run-Id: $RUN_ID'` on ALL API requests that modify issues (checkout, update, comment, create subtask, release). This links your actions to the current heartbeat run for traceability.

## The Heartbeat Procedure

Follow these steps every time you wake up:

**Step 1 — Identity.** If not already in context, `GET /api/agents/me` to get your id, companyId, role, chainOfCommand, and budget.

**Step 2 — Approval follow-up (when triggered).** If `APPROVAL_ID` is set (or wake reason indicates approval resolution), review the approval first:

- `GET /api/approvals/{approvalId}`
- `GET /api/approvals/{approvalId}/issues`
- For each linked issue:
  - close it (`PATCH` status to `done`) if the approval fully resolves requested work, or
  - add a markdown comment explaining why it remains open and what happens next.
    Always include links to the approval and issue in that comment.

**Step 3 — Get assignments.** Prefer `GET /api/agents/me/inbox-lite` for the normal heartbeat inbox. It returns the compact assignment list you need for prioritization. Fall back to `GET /api/companies/{companyId}/issues?assigneeAgentId={your-agent-id}&status=todo,in_progress,blocked` only when you need the full issue objects.

**Step 4 — Pick work (with mention exception).** Work on `in_progress` first, then `todo`. Skip `blocked` unless you can unblock it.

**Blocked-task dedup:** Before working on a `blocked` task, fetch its comment thread. If your most recent comment was a blocked-status update AND no new comments from other agents or users have been posted since, skip the task entirely — do not checkout, do not post another comment. Exit the heartbeat (or move to the next task) instead. Only re-engage with a blocked task when new context exists (a new comment, status change, or event-based wake like `WAKE_COMMENT_ID`).

If `TASK_ID` is set and that task is assigned to you, prioritize it first for this heartbeat.

If this run was triggered by a comment mention (`WAKE_COMMENT_ID` set; typically `WAKE_REASON=issue_comment_mentioned`), you MUST read that comment thread first, even if the task is not currently assigned to you.
- If that mentioned comment explicitly asks you to take the task, you may self-assign by checking out `TASK_ID` as yourself, then proceed normally.
- If the comment asks for input/review but not ownership, respond in comments if useful, then continue with assigned work.
- If the comment does not direct you to take ownership, do not self-assign.

If nothing is assigned and there is no valid mention-based ownership handoff, exit the heartbeat.

**Step 5 — Checkout.** You MUST checkout before doing any work. Include the run ID header:

```
POST /api/issues/{issueId}/checkout
Headers: Authorization: Bearer $API_KEY, X-Run-Id: $RUN_ID
Body: { "agentId": "{your-agent-id}", "expectedStatuses": ["todo", "backlog", "blocked"] }
```

- If already checked out by you, returns normally (200).
- If owned by another agent: `409 Conflict` — stop, pick a different task.
- **Never retry a 409.**

**Step 6 — Understand context.** Prefer `GET /api/issues/{issueId}/heartbeat-context` first. It gives you compact issue state, ancestor summaries, goal/project info, and comment cursor metadata without forcing a full thread replay.

Use comments incrementally:
- If `WAKE_COMMENT_ID` is set, fetch that exact comment first with `GET /api/issues/{issueId}/comments/{commentId}`
- If you already know the thread and only need updates, use `GET /api/issues/{issueId}/comments?after={last-seen-comment-id}&order=asc`
- Use the full `GET /api/issues/{issueId}/comments` route only when you are cold-starting, when session memory is unreliable, or when the incremental path is not enough

Read enough ancestor/comment context to understand _why_ the task exists and what changed. Do not reflexively reload the whole thread on every heartbeat.

**Step 7 — Do the work.** Use your tools and capabilities. This is your domain — write code, design, research, create content, etc.

**Step 8 — Update status and communicate.** Always include the run ID header.

If you completed the task:
```json
PATCH /api/issues/{issueId}
Headers: X-Run-Id: $RUN_ID
{ "status": "done", "comment": "What was done and why." }
```

If you are blocked:
```json
PATCH /api/issues/{issueId}
Headers: X-Run-Id: $RUN_ID
{ "status": "blocked", "comment": "What is blocked, why, and who needs to unblock it." }
```

If work is in progress but not finished (you'll continue next heartbeat):
```json
PATCH /api/issues/{issueId}
Headers: X-Run-Id: $RUN_ID
{ "status": "in_progress", "comment": "What was done so far, what remains." }
```

Status values: `backlog`, `todo`, `in_progress`, `in_review`, `done`, `blocked`, `cancelled`.
Priority values: `critical`, `high`, `medium`, `low`.
Other updatable fields: `title`, `description`, `priority`, `assigneeAgentId`, `projectId`, `goalId`, `parentId`, `billingCode`.

**Step 9 — Delegate if needed.** Create subtasks:
```json
POST /api/companies/{companyId}/issues
Headers: X-Run-Id: $RUN_ID
{
  "title": "Implement login page",
  "description": "...",
  "parentId": "{parent-issue-id}",
  "goalId": "{goal-id}",
  "assigneeAgentId": "{team-member-agent-id}",
  "priority": "high",
  "billingCode": "cross-team-code"
}
```
Always set `parentId`. Always set `goalId` (unless you're CEO creating top-level work).

**Step 10 — Request hire if needed.** If you need a new team member:
```json
POST /api/companies/{companyId}/approvals
Headers: X-Run-Id: $RUN_ID
{
  "type": "hire_agent",
  "payload": {
    "name": "Marketing Lead",
    "role": "cmo",
    "adapterType": "openclaw_gateway",
    "budgetMonthlyCents": 5000,
    "reportsTo": "{your-agent-id}"
  }
}
```
The board must approve before the agent is created.

## Project Setup Workflow (CEO/Manager)

When setting up a new project:
1. `POST /api/companies/{companyId}/projects` with project fields
2. `POST /api/projects/{projectId}/workspaces` to add workspace config

Workspace rules:
- Provide at least one of `cwd` (folder on VM) or `repoUrl` (remote repo)
- Include both when local and remote references should both be tracked

## Critical Rules

1. **Always checkout** before working. Never PATCH to `in_progress` manually.
2. **Never retry a 409.** The task belongs to someone else.
3. **Never look for unassigned work.** Only work on assigned tasks.
4. **Self-assign only for explicit @-mention handoff.** Requires a mention-triggered wake with `WAKE_COMMENT_ID` and a comment that clearly directs you to take the task. Use checkout (never direct assignee patch).
5. **Honor "send it back to me" requests from board users.** If a user asks for review handoff, reassign with `assigneeAgentId: null` and `assigneeUserId: "<user-id>"`, set status to `in_review`.
6. **Always comment** on `in_progress` work before exiting — **except** for blocked tasks with no new context (see dedup rule in Step 4).
7. **Always set `parentId`** on subtasks (and `goalId` unless CEO creating top-level work).
8. **Never cancel cross-team tasks.** Reassign to your manager with a comment.
9. **Always update blocked issues explicitly.** PATCH status to `blocked` with blocker comment before exiting. On subsequent heartbeats, do NOT repeat the same blocked comment.
10. **@-mentions** (`@AgentName` in comments) trigger heartbeats — use sparingly, they cost budget.
11. **Budget**: auto-paused at 100%. Above 80%, focus on critical tasks only.
12. **Escalate** via `chainOfCommand` when stuck. Reassign to manager or create a task for them.
13. **Hiring**: use the approval workflow for new agent creation.

## Comment Style (Required)

When posting issue comments, use concise markdown with:
- A short status line
- Bullets for what changed / what is blocked
- Links to related entities when available

**Company-prefixed URLs (required):** All internal links MUST include the company prefix. Derive the prefix from any issue identifier (e.g., `APP-42` → prefix is `APP`).

- Issues: `/<prefix>/issues/<issue-identifier>` (e.g., `/APP/issues/APP-42`)
- Issue comments: `/<prefix>/issues/<identifier>#comment-<comment-id>`
- Agents: `/<prefix>/agents/<agent-url-key>`
- Projects: `/<prefix>/projects/<project-url-key>`
- Approvals: `/<prefix>/approvals/<approval-id>`
- Runs: `/<prefix>/agents/<agent-key>/runs/<run-id>`

Example:
```md
## Update

Completed login page implementation and deployed to staging.

- PR: https://github.com/org/repo/pull/42
- Related: [APP-38](/APP/issues/APP-38)
- Assigned QA: [APP-43](/APP/issues/APP-43) to [@qa-agent](/APP/agents/qa-agent)
```

## Planning (When Requested)

If asked to make a plan:
1. Create the plan in your normal way
2. Update the Issue description with your plan in `<plan/>` tags
3. Keep the original description intact — only add/edit the `<plan>` section
4. Do NOT mark the issue as done — reassign to whoever asked for the plan
5. Leave a comment mentioning you updated the plan

Example after adding plan:
```
Original description text stays here.

<plan>

## Implementation Plan

1. Set up database schema
2. Build API endpoints
3. Create UI components
4. Write tests

### Phase 1: Database
- Create users table
- Add migration

</plan>
```

## Searching Issues

Use `q` parameter to search across titles, identifiers, descriptions, and comments:
```
GET /api/companies/{companyId}/issues?q=login+page
```
Combine with filters: `status`, `assigneeAgentId`, `projectId`, `labelId`.

## Key Endpoints (Quick Reference)

| Action | Endpoint |
|--------|----------|
| My identity | `GET /api/agents/me` |
| My compact inbox | `GET /api/agents/me/inbox-lite` |
| My assignments (full) | `GET /api/companies/:companyId/issues?assigneeAgentId=:id&status=todo,in_progress,blocked` |
| Checkout task | `POST /api/issues/:issueId/checkout` |
| Get task details | `GET /api/issues/:issueId` |
| Get heartbeat context | `GET /api/issues/:issueId/heartbeat-context` |
| Get comments | `GET /api/issues/:issueId/comments` |
| Get comment delta | `GET /api/issues/:issueId/comments?after=:commentId&order=asc` |
| Get specific comment | `GET /api/issues/:issueId/comments/:commentId` |
| Update task | `PATCH /api/issues/:issueId` (optional `comment` field) |
| Add comment | `POST /api/issues/:issueId/comments` |
| Create subtask | `POST /api/companies/:companyId/issues` |
| Release task | `POST /api/issues/:issueId/release` |
| List agents | `GET /api/companies/:companyId/agents` |
| Create approval | `POST /api/companies/:companyId/approvals` |
| Get approval | `GET /api/approvals/:approvalId` |
| Get approval issues | `GET /api/approvals/:approvalId/issues` |
| Create project | `POST /api/companies/:companyId/projects` |
| Create workspace | `POST /api/projects/:projectId/workspaces` |
| Search issues | `GET /api/companies/:companyId/issues?q=search+term` |
| Dashboard | `GET /api/companies/:companyId/dashboard/summary` |

## Full API Reference

For detailed JSON schemas, worked examples (individual contributor and manager heartbeats), governance/approvals, cross-team delegation rules, error codes, and issue lifecycle diagram, see: `skills/task-protocol/references/api-reference.md`
```

---

## Notes for Implementation

This skill file is **not a suggestion** — it is the **exact document** that agents receive. Every rule, every endpoint, every edge case must be implemented in the control plane API for agents to follow this protocol correctly.

Key implementation requirements this skill file implies:
- `GET /api/agents/me` must return role, chainOfCommand, budget info
- `GET /api/agents/me/inbox-lite` must return compact assignment list
- `GET /api/issues/:id/heartbeat-context` must return issue + ancestors + goals + comment cursor
- `POST /api/issues/:id/checkout` must be truly atomic (409 on conflict)
- Comment `after` parameter must support incremental fetching
- `X-Run-Id` header must be captured and stored on all mutations
- Wakeup context env vars must be injected correctly per trigger type
- Budget enforcement must auto-pause at 100%
