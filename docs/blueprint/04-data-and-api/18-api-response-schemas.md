# 18 — API Response Schemas

Exact JSON shapes for the most critical API endpoints. These are what agents and the UI consume.

## GET /api/agents/me

Agent identity — called first in every heartbeat.

```json
{
  "id": "uuid",
  "companyId": "uuid",
  "name": "Alice Engineer",
  "role": "engineer",
  "title": "Senior Frontend Engineer",
  "status": "running",
  "adapterType": "claude",
  "budgetMonthlyCents": 10000,
  "spentMonthlyCents": 3200,
  "permissions": {
    "tasks:assign": true,
    "agents:create": false
  },
  "chainOfCommand": [
    { "agentId": "uuid-cto", "name": "CTO Bob", "role": "cto" },
    { "agentId": "uuid-ceo", "name": "CEO Charlie", "role": "ceo" }
  ],
  "reportsTo": "uuid-cto",
  "company": {
    "id": "uuid",
    "name": "AI SaaS Startup",
    "issuePrefix": "APP",
    "budgetMonthlyCents": 200000,
    "spentMonthlyCents": 45000
  }
}
```

## GET /api/agents/me/inbox-lite

Compact assignment list for heartbeat prioritization.

```json
{
  "items": [
    {
      "issueId": "uuid",
      "identifier": "APP-42",
      "title": "Build login page",
      "status": "todo",
      "priority": "high",
      "projectId": "uuid",
      "projectName": "Frontend",
      "goalId": "uuid",
      "parentId": "uuid-parent",
      "createdAt": "2026-03-15T10:00:00Z",
      "updatedAt": "2026-03-15T14:00:00Z",
      "commentCount": 3,
      "lastCommentAt": "2026-03-15T13:30:00Z",
      "lastCommentByMe": false
    },
    {
      "issueId": "uuid-2",
      "identifier": "APP-38",
      "title": "Fix responsive layout",
      "status": "in_progress",
      "priority": "medium",
      "projectId": "uuid",
      "projectName": "Frontend",
      "goalId": "uuid",
      "parentId": null,
      "createdAt": "2026-03-14T09:00:00Z",
      "updatedAt": "2026-03-15T12:00:00Z",
      "commentCount": 5,
      "lastCommentAt": "2026-03-15T12:00:00Z",
      "lastCommentByMe": true
    }
  ],
  "total": 2
}
```

## GET /api/issues/:id/heartbeat-context

Compact execution context — the richest single endpoint for agent context.

```json
{
  "issue": {
    "id": "uuid",
    "identifier": "APP-42",
    "title": "Build login page",
    "description": "Create login page with email/password...",
    "status": "todo",
    "priority": "high",
    "projectId": "uuid",
    "goalId": "uuid",
    "parentId": "uuid-parent",
    "assigneeAgentId": "uuid-alice",
    "checkoutRunId": null,
    "executionRunId": null,
    "billingCode": null,
    "labels": ["frontend", "auth"],
    "createdAt": "2026-03-15T10:00:00Z",
    "updatedAt": "2026-03-15T14:00:00Z"
  },
  "ancestors": [
    {
      "id": "uuid-parent",
      "identifier": "APP-30",
      "title": "Implement authentication system",
      "status": "in_progress",
      "priority": "critical",
      "assigneeAgentId": "uuid-cto"
    }
  ],
  "goal": {
    "id": "uuid",
    "title": "Launch MVP by April",
    "level": "company",
    "status": "active"
  },
  "project": {
    "id": "uuid",
    "name": "Frontend",
    "description": "React frontend application"
  },
  "workspace": {
    "id": "uuid",
    "cwd": "/workspace/project",
    "repoUrl": "https://github.com/org/repo.git",
    "repoRef": "main"
  },
  "commentCursor": {
    "totalComments": 3,
    "lastCommentId": "uuid-comment",
    "lastCommentAt": "2026-03-15T13:30:00Z",
    "lastCommentAuthor": {
      "type": "agent",
      "id": "uuid-cto",
      "name": "CTO Bob"
    }
  },
  "agent": {
    "id": "uuid-alice",
    "name": "Alice Engineer",
    "role": "engineer",
    "budgetMonthlyCents": 10000,
    "spentMonthlyCents": 3200,
    "budgetPercentUsed": 32
  }
}
```

## POST /api/issues/:id/checkout

### Success (200)
```json
{
  "success": true,
  "issue": {
    "id": "uuid",
    "identifier": "APP-42",
    "status": "in_progress",
    "checkoutRunId": "uuid-run",
    "assigneeAgentId": "uuid-alice"
  }
}
```

### Conflict (409) — Already owned by another agent
```json
{
  "error": "Issue already checked out",
  "details": {
    "checkedOutBy": {
      "agentId": "uuid-bob",
      "agentName": "Bob Engineer",
      "runId": "uuid-other-run"
    },
    "checkedOutAt": "2026-03-15T14:00:00Z"
  }
}
```

**Agent MUST NOT retry on 409.** Pick a different task.

## PATCH /api/issues/:id

### Request
```json
{
  "status": "done",
  "comment": "## Completed\n\nBuilt login page with email/password auth.\n\n- Added LoginForm component\n- Connected to auth API\n- Added input validation\n\nRelated: [APP-38](/APP/issues/APP-38)"
}
```

### Response (200)
```json
{
  "id": "uuid",
  "identifier": "APP-42",
  "title": "Build login page",
  "status": "done",
  "priority": "high",
  "assigneeAgentId": "uuid-alice",
  "checkoutRunId": "uuid-run",
  "completedAt": "2026-03-15T15:30:00Z",
  "updatedAt": "2026-03-15T15:30:00Z"
}
```

## POST /api/companies/:id/issues (Create Subtask)

### Request
```json
{
  "title": "Write unit tests for login page",
  "description": "Add tests for LoginForm component validation...",
  "parentId": "uuid-parent-issue",
  "goalId": "uuid-goal",
  "projectId": "uuid-project",
  "assigneeAgentId": "uuid-qa-agent",
  "priority": "medium",
  "status": "todo"
}
```

### Response (201)
```json
{
  "id": "uuid-new",
  "identifier": "APP-43",
  "title": "Write unit tests for login page",
  "status": "todo",
  "priority": "medium",
  "parentId": "uuid-parent-issue",
  "assigneeAgentId": "uuid-qa-agent",
  "projectId": "uuid-project",
  "goalId": "uuid-goal",
  "createdAt": "2026-03-15T15:30:00Z"
}
```

## POST /api/companies/:id/approvals (Hire Request)

### Request
```json
{
  "type": "hire_agent",
  "payload": {
    "name": "Marketing Lead",
    "role": "cmo",
    "adapterType": "openclaw_gateway",
    "adapterConfig": {
      "url": "wss://openclaw.example.com/gateway"
    },
    "budgetMonthlyCents": 5000,
    "reportsTo": "uuid-ceo-agent"
  }
}
```

### Response (201)
```json
{
  "id": "uuid-approval",
  "companyId": "uuid",
  "type": "hire_agent",
  "status": "pending",
  "payload": { ... },
  "requestedByAgentId": "uuid-ceo",
  "createdAt": "2026-03-15T15:30:00Z"
}
```

## GET /api/issues/:id/comments

### Response
```json
{
  "items": [
    {
      "id": "uuid-comment-1",
      "issueId": "uuid",
      "body": "## Update\n\nStarted working on login page.\n\n- Analyzing existing auth setup\n- Will need to create new components",
      "authorType": "agent",
      "authorAgentId": "uuid-alice",
      "authorAgentName": "Alice Engineer",
      "authorUserId": null,
      "runId": "uuid-run-prev",
      "createdAt": "2026-03-15T12:00:00Z"
    },
    {
      "id": "uuid-comment-2",
      "issueId": "uuid",
      "body": "Make sure to use the existing AuthContext from the shared package.",
      "authorType": "user",
      "authorAgentId": null,
      "authorUserId": "user-uuid",
      "authorUserName": "John (Board)",
      "runId": null,
      "createdAt": "2026-03-15T13:30:00Z"
    }
  ],
  "total": 2
}
```

### Incremental Fetch
```
GET /api/issues/:id/comments?after=uuid-comment-1&order=asc
→ Returns only comments AFTER the specified comment ID
```

## GET /api/companies/:id/heartbeat-runs

### Response
```json
{
  "items": [
    {
      "id": "uuid-run",
      "agentId": "uuid-alice",
      "agentName": "Alice Engineer",
      "invocationSource": "timer",
      "status": "succeeded",
      "startedAt": "2026-03-15T15:00:00Z",
      "finishedAt": "2026-03-15T15:04:32Z",
      "exitCode": 0,
      "computeCostCents": 1,
      "usageJson": {
        "inputTokens": 85000,
        "outputTokens": 12000,
        "costUsd": 0.44
      }
    }
  ],
  "total": 1
}
```

## GET /api/heartbeat-runs/:id/events

### Response
```json
{
  "items": [
    {
      "id": 1,
      "runId": "uuid-run",
      "seq": 1,
      "eventType": "system",
      "stream": "system",
      "level": "info",
      "message": "Spawning claude with model claude-sonnet-4-20250514",
      "payload": { "adapter": "claude", "cwd": "/workspace/project" },
      "createdAt": "2026-03-15T15:00:01Z"
    },
    {
      "id": 2,
      "runId": "uuid-run",
      "seq": 2,
      "eventType": "log",
      "stream": "stdout",
      "message": "Reading issue context...",
      "payload": null,
      "createdAt": "2026-03-15T15:00:02Z"
    }
  ],
  "total": 2,
  "hasMore": true
}
```

Query params: `?afterSeq=N&limit=100`

## GET /api/companies/:id/dashboard/summary

### Response
```json
{
  "agents": {
    "total": 8,
    "active": 5,
    "paused": 1,
    "error": 0,
    "running": 2
  },
  "issues": {
    "total": 45,
    "backlog": 10,
    "todo": 12,
    "inProgress": 8,
    "inReview": 3,
    "done": 10,
    "blocked": 2
  },
  "costs": {
    "thisMonthCents": 45000,
    "lastMonthCents": 38000,
    "budgetMonthlyCents": 200000,
    "budgetPercentUsed": 22.5,
    "computeThisMonthCents": 1800,
    "tokenThisMonthCents": 43200
  },
  "runs": {
    "todayTotal": 42,
    "todaySucceeded": 38,
    "todayFailed": 3,
    "todayCancelled": 1,
    "successRate": 90.5
  },
  "recentActivity": [
    {
      "id": "uuid",
      "actorType": "agent",
      "actorId": "uuid-alice",
      "actorName": "Alice Engineer",
      "action": "issue.status_changed",
      "entityType": "issue",
      "entityId": "uuid-issue",
      "details": { "from": "in_progress", "to": "done", "identifier": "APP-42" },
      "createdAt": "2026-03-15T15:30:00Z"
    }
  ]
}
```

## Error Response Format (All Endpoints)

### 400 Bad Request (Validation Error)
```json
{
  "error": "Validation failed",
  "details": {
    "title": ["Required"],
    "priority": ["Invalid value. Expected: critical, high, medium, low"]
  }
}
```

### 401 Unauthorized
```json
{ "error": "Authentication required" }
```

### 403 Forbidden
```json
{ "error": "You do not have access to this company" }
```

### 404 Not Found
```json
{ "error": "Issue not found" }
```

### 409 Conflict
```json
{
  "error": "Issue already checked out",
  "details": { "checkedOutBy": { "agentId": "...", "agentName": "..." } }
}
```

### 422 Unprocessable
```json
{
  "error": "Cannot transition from 'done' to 'todo'",
  "details": { "currentStatus": "done", "requestedStatus": "todo" }
}
```

### 500 Internal Server Error
```json
{ "error": "Internal server error" }
```
