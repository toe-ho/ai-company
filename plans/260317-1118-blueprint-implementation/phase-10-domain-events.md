# Phase 10: Domain Events

## Context Links
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md) (events/ directory)
- [20 - Background Jobs](../../docs/blueprint/05-operations/20-background-jobs-and-async-processing.md)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 2h
- **Description:** Domain event definitions and handlers for side effects: auto-create agent on hire approval, auto-pause on budget exceeded, trigger wakeups on issue assignment/mention.

## Key Insights
- Events are published via `@nestjs/cqrs` EventBus
- Event handlers run asynchronously after the command completes
- Events are fire-and-forget; failure in handler should not affect command
- `OnApprovalResolved` is the most complex: auto-creates agent if hire_agent approved
- Live events (Redis pub/sub) are separate from domain events (internal NestJS)

## Architecture
```
apps/backend/src/application/events/
├── agent-status-changed-event.ts
├── issue-checked-out-event.ts
├── issue-status-changed-event.ts
├── heartbeat-run-completed-event.ts
├── approval-resolved-event.ts
├── agent-hired-event.ts
├── budget-exceeded-event.ts
├── handlers/
│   ├── on-approval-resolved-handler.ts
│   ├── on-heartbeat-completed-handler.ts
│   ├── on-budget-exceeded-handler.ts
│   ├── on-issue-assigned-handler.ts
│   └── on-agent-mentioned-handler.ts
└── index.ts

apps/backend/src/application/contexts/
└── actor/
    ├── actor-context-service.ts
    └── index.ts
```

## Related Code Files

### Files to Create
- 7 event definition files
- 5 event handler files
- ActorContextService
- Barrel exports

## Implementation Steps

1. **Create event definitions**
   ```typescript
   // Each event is a plain class
   export class AgentStatusChangedEvent {
     constructor(
       public readonly companyId: string,
       public readonly agentId: string,
       public readonly previousStatus: string,
       public readonly newStatus: string,
     ) {}
   }
   ```
   - `IssueCheckedOutEvent(companyId, issueId, agentId)`
   - `IssueStatusChangedEvent(companyId, issueId, from, to)`
   - `HeartbeatRunCompletedEvent(companyId, agentId, runId, status, usage)`
   - `ApprovalResolvedEvent(companyId, approvalId, type, status, payload)`
   - `AgentHiredEvent(companyId, agentId, hiredByAgentId)`
   - `BudgetExceededEvent(companyId, agentId, budgetCents, spentCents)`

2. **Create `OnApprovalResolved` handler**
   - Listens for `ApprovalResolvedEvent`
   - If type='hire_agent' AND status='approved':
     1. Extract agent config from approval.payload
     2. Execute `CreateAgentCommand` via commandBus
     3. Publish `AgentHiredEvent`
     4. If adapter has `onHireApproved` hook, call it
   - Publish live event for UI notification

3. **Create `OnHeartbeatCompleted` handler**
   - Listens for `HeartbeatRunCompletedEvent`
   - Update `AgentRuntimeState` (totalTokens, totalCost, lastHeartbeatAt)
   - If run succeeded: set agent status to 'idle'
   - If run failed 3+ consecutive times: set agent status to 'error'
   - Publish live event

4. **Create `OnBudgetExceeded` handler**
   - Listens for `BudgetExceededEvent`
   - Set agent status to 'paused'
   - Log activity: 'agent.budget_exceeded'
   - Publish live event (for UI toast notification)

5. **Create `OnIssueAssigned` handler**
   - Listens for issue assignment (when assigneeAgentId changes)
   - Enqueue wakeup request for assigned agent: `WakeupAgentCommand(agentId, source='assignment', taskId)`

6. **Create `OnAgentMentioned` handler**
   - Listens for @mention detection in comments
   - Enqueue wakeup request for mentioned agent: `WakeupAgentCommand(agentId, source='on_demand', commentId)`

7. **Create `ActorContextService`**
   - Request-scoped service
   - Resolves current actor from request context
   - Used by interceptors and handlers that need actor info outside controller scope

8. **Create `index.ts`** exporting `EventHandlers` array

## Todo List
- [ ] 7 event definition classes
- [ ] OnApprovalResolved handler (auto-hire)
- [ ] OnHeartbeatCompleted handler (update runtime state)
- [ ] OnBudgetExceeded handler (auto-pause)
- [ ] OnIssueAssigned handler (trigger wakeup)
- [ ] OnAgentMentioned handler (trigger wakeup)
- [ ] ActorContextService
- [ ] EventHandlers export array
- [ ] typecheck passes

## Success Criteria
- Approving hire_agent approval auto-creates agent
- Budget exceeded auto-pauses agent
- Issue assignment triggers agent wakeup
- All events publish to Redis for live UI updates
- Event handler failures don't crash the command

## Risk Assessment
- **Circular dependency:** Event handler calls commandBus which may re-trigger events. Use `onModuleInit` registration.
- **Async error handling:** Wrap handler bodies in try/catch; log errors but don't propagate

## Security Considerations
- Event handlers respect same company scope as commands
- No PII in event payloads (use IDs, not names)

## Next Steps
- Phase 11: Register EventHandlers in SharedModule
