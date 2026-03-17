# Phase 8: CQRS Handlers (Core)

## Context Links
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md) (commands/ + queries/ directory)
- [11 - Backend Architecture](../../docs/blueprint/03-architecture/11-backend-architecture.md) (heartbeat service)
- [18 - API Response Schemas](../../docs/blueprint/04-data-and-api/18-api-response-schemas.md)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 4h
- **Description:** Implement core CQRS command and query handlers for Company, Agent, Issue, and Heartbeat domains.

## Key Insights
- Each handler is a separate class: `XyzCommand.ts` (data class) + `XyzHandler.ts` (logic)
- Handlers inject repositories and services via `@Inject('IXyzRepository')` tokens
- Commands mutate state; queries are read-only
- `InvokeHeartbeatHandler` is the most complex handler (~100 lines), orchestrates entire execution flow
- `CheckoutIssueHandler` uses atomic checkout; returns 409 on conflict
- Commands publish domain events via EventBus for side effects

## Requirements
### Functional
- Company: CRUD + create-from-template
- Agent: CRUD + pause/resume/terminate + config rollback + API key management
- Issue: CRUD + atomic checkout/release + comments + attachments
- Heartbeat: invoke, wakeup, cancel, reap orphaned

### Non-functional
- All handlers are @Injectable NestJS providers
- Type-safe command/query payloads

## Architecture
```
apps/backend/src/application/
├── commands/
│   ├── company/
│   │   ├── create-company-command.ts
│   │   ├── create-company-handler.ts
│   │   ├── create-company-from-template-command.ts
│   │   ├── create-company-from-template-handler.ts
│   │   ├── update-company-command.ts
│   │   ├── update-company-handler.ts
│   │   ├── delete-company-command.ts
│   │   └── delete-company-handler.ts
│   ├── agent/
│   │   ├── create-agent-command.ts
│   │   ├── create-agent-handler.ts
│   │   ├── update-agent-command.ts
│   │   ├── update-agent-handler.ts
│   │   ├── pause-agent-command.ts
│   │   ├── pause-agent-handler.ts
│   │   ├── resume-agent-command.ts
│   │   ├── resume-agent-handler.ts
│   │   ├── terminate-agent-command.ts
│   │   ├── terminate-agent-handler.ts
│   │   ├── rollback-agent-config-command.ts
│   │   ├── rollback-agent-config-handler.ts
│   │   ├── create-agent-api-key-command.ts
│   │   ├── create-agent-api-key-handler.ts
│   │   ├── revoke-agent-api-key-command.ts
│   │   └── revoke-agent-api-key-handler.ts
│   ├── issue/
│   │   ├── create-issue-command.ts
│   │   ├── create-issue-handler.ts
│   │   ├── update-issue-command.ts
│   │   ├── update-issue-handler.ts
│   │   ├── checkout-issue-command.ts
│   │   ├── checkout-issue-handler.ts
│   │   ├── release-issue-command.ts
│   │   ├── release-issue-handler.ts
│   │   ├── add-comment-command.ts
│   │   ├── add-comment-handler.ts
│   │   ├── upload-attachment-command.ts
│   │   └── upload-attachment-handler.ts
│   └── heartbeat/
│       ├── invoke-heartbeat-command.ts
│       ├── invoke-heartbeat-handler.ts
│       ├── wakeup-agent-command.ts
│       ├── wakeup-agent-handler.ts
│       ├── cancel-run-command.ts
│       ├── cancel-run-handler.ts
│       ├── reap-orphaned-runs-command.ts
│       └── reap-orphaned-runs-handler.ts
├── queries/
│   ├── company/
│   │   ├── get-company-query.ts
│   │   ├── get-company-handler.ts
│   │   ├── list-companies-query.ts
│   │   └── list-companies-handler.ts
│   ├── agent/
│   │   ├── get-agent-query.ts
│   │   ├── get-agent-handler.ts
│   │   ├── list-agents-query.ts
│   │   ├── list-agents-handler.ts
│   │   ├── get-agent-me-query.ts
│   │   ├── get-agent-me-handler.ts
│   │   ├── get-agent-inbox-query.ts
│   │   ├── get-agent-inbox-handler.ts
│   │   ├── get-org-tree-query.ts
│   │   ├── get-org-tree-handler.ts
│   │   ├── list-config-revisions-query.ts
│   │   ├── list-config-revisions-handler.ts
│   │   ├── get-runtime-state-query.ts
│   │   └── get-runtime-state-handler.ts
│   ├── issue/
│   │   ├── get-issue-query.ts
│   │   ├── get-issue-handler.ts
│   │   ├── list-issues-query.ts
│   │   ├── list-issues-handler.ts
│   │   ├── get-heartbeat-context-query.ts
│   │   ├── get-heartbeat-context-handler.ts
│   │   ├── list-comments-query.ts
│   │   ├── list-comments-handler.ts
│   │   ├── search-issues-query.ts
│   │   └── search-issues-handler.ts
│   └── heartbeat/
│       ├── list-runs-query.ts
│       ├── list-runs-handler.ts
│       ├── get-run-query.ts
│       ├── get-run-handler.ts
│       ├── list-run-events-query.ts
│       ├── list-run-events-handler.ts
│       ├── get-live-runs-query.ts
│       └── get-live-runs-handler.ts
└── index.ts                      # Export arrays: CommandHandlers, QueryHandlers
```

## Related Code Files

### Files to Create
~70 files (command/query classes + handlers).

## Implementation Steps

1. **Establish command/query pattern**
   ```typescript
   // Command data class
   export class CreateCompanyCommand {
     constructor(
       public readonly userId: string,
       public readonly data: { name: string; description?: string; templateId?: string },
     ) {}
   }

   // Handler
   @CommandHandler(CreateCompanyCommand)
   export class CreateCompanyHandler implements ICommandHandler<CreateCompanyCommand> {
     constructor(
       @Inject('ICompanyRepository') private readonly companyRepo: ICompanyRepository,
       @Inject('IUserCompanyRepository') private readonly userCompanyRepo: any,
       private readonly eventBus: EventBus,
     ) {}
     async execute(cmd: CreateCompanyCommand) { /* ... */ }
   }
   ```

2. **Company commands**
   - `CreateCompany`: Create company + userCompanies entry (role: owner)
   - `CreateCompanyFromTemplate`: Load template -> create company -> create agents from template.agentConfigs -> create goal from template.goalTemplate
   - `UpdateCompany`: Partial update
   - `DeleteCompany`: Cascade destroy VM, archive agents

3. **Agent commands**
   - `CreateAgent`: Validate company exists, create agent + initial configRevision + runtimeState
   - `UpdateAgent`: Update fields, create new configRevision if adapter/runtime config changed
   - `PauseAgent`: Set status='paused', cancel any running heartbeat
   - `ResumeAgent`: Set status='active'
   - `TerminateAgent`: Set status='terminated', destroy VM if last agent in company
   - `RollbackAgentConfig`: Load revision, apply as current config
   - `CreateAgentApiKey`: Generate `pcp_` + 32 random chars, hash with SHA-256, store hash, return plaintext once
   - `RevokeAgentApiKey`: Set revokedAt

4. **Issue commands**
   - `CreateIssue`: Generate identifier (prefix + counter), create issue, publish IssueCreatedEvent, if assigneeAgentId -> publish OnIssueAssigned
   - `UpdateIssue`: Validate status transitions, update fields, log activity
   - `CheckoutIssue`: Per blueprint 12 example code. Validate status -> atomicCheckout -> publish event -> log activity. Throw ConflictException on lock failure.
   - `ReleaseIssue`: Clear checkoutRunId, set status back to 'todo'
   - `AddComment`: Create issueComment, check for @mentions -> publish OnAgentMentioned
   - `UploadAttachment`: Upload to S3 via IStorageService -> create asset + issueAttachment records

5. **Heartbeat commands** (CRITICAL)
   - `InvokeHeartbeat`: The big orchestrator per blueprint 11:
     1. Load agent, validate status=active, check budget
     2. Create heartbeatRun (status: queued)
     3. Retrieve API key via IApiKeyVaultService
     4. Boot VM via commandBus.execute(EnsureVmCommand)
     5. Build IExecutionRequest (agent context, session, workspace, API key, JWT)
     6. Call IExecutionEngineService.execute(request) -> iterate SSE stream
     7. For each event: create HeartbeatRunEvent, publish live event
     8. On completion: RecordCostEvent, update AgentRuntimeState, update Agent.spentMonthlyCents
     9. If over budget -> publish BudgetExceededEvent
     10. Start idle timer for VM hibernate
   - `WakeupAgent`: Enqueue wakeup request, coalesce duplicates
   - `CancelRun`: Send cancel to executor, update run status
   - `ReapOrphanedRuns`: Find runs with status='running' and updatedAt < 5min ago, set to 'failed'

6. **Company queries**
   - `ListCompanies`: By userId via userCompanies junction
   - `GetCompany`: By id + verify access

7. **Agent queries**
   - `ListAgents`: By companyId with optional status filter
   - `GetAgent`: By id + company scope
   - `GetAgentMe`: By agentId from actor context (agent self-identity)
   - `GetAgentInbox`: Issues assigned to agent, ordered by priority/updated
   - `GetOrgTree`: Recursive tree from AgentRepository
   - `ListConfigRevisions`: By agentId
   - `GetRuntimeState`: By agentId

8. **Issue queries**
   - `ListIssues`: By companyId with status/assignee/priority filters
   - `GetIssue`: By id + company scope
   - `GetHeartbeatContext`: Rich query joining issue + ancestors + goal + project + workspace + comments (per blueprint 18)
   - `ListComments`: By issueId with cursor pagination (afterId)
   - `SearchIssues`: ILIKE on title/description

9. **Heartbeat queries**
   - `ListRuns`: By companyId or agentId, ordered by startedAt desc
   - `GetRun`: By id
   - `ListRunEvents`: By runId, cursor by seq, limit
   - `GetLiveRuns`: Currently running runs for company

10. **Create `index.ts`** exporting `CommandHandlers` and `QueryHandlers` arrays

## Todo List
- [ ] Company commands (4) + queries (2)
- [ ] Agent commands (8) + queries (7)
- [ ] Issue commands (6) + queries (5)
- [ ] Heartbeat commands (4) + queries (4)
- [ ] InvokeHeartbeatHandler (complex orchestration)
- [ ] CheckoutIssueHandler (atomic lock)
- [ ] Handler export arrays
- [ ] typecheck passes

## Success Criteria
- All ~35 command handlers and ~18 query handlers created
- InvokeHeartbeatHandler orchestrates full execution flow
- CheckoutIssueHandler correctly handles atomic lock + 409
- All handlers use @Inject tokens for repositories/services
- `turbo typecheck` passes

## Risk Assessment
- **InvokeHeartbeat complexity:** Keep handler focused on orchestration; delegate details to services
- **SSE stream error handling:** Wrap in try/catch; set run to 'failed' on error (don't re-throw)
- **Command handler registration:** Must list ALL handlers in SharedModule providers

## Security Considerations
- Commands verify actor has permission (company membership)
- Agent API keys: plaintext returned once only on CreateAgentApiKey
- API key never logged or included in error responses
- Budget checks before execution to prevent runaway costs

## Next Steps
- Phase 9: Supporting CQRS handlers (goals, projects, approvals, cost, etc.)
- Phase 10: Domain events triggered by these handlers
