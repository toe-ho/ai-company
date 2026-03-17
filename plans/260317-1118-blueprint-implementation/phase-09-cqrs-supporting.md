# Phase 9: CQRS Handlers (Supporting)

## Context Links
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md)
- [17 - API Design](../../docs/blueprint/04-data-and-api/17-api-design.md)
- [18 - API Response Schemas](../../docs/blueprint/04-data-and-api/18-api-response-schemas.md)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 3h
- **Description:** Supporting CQRS handlers for goals, projects, approvals, cost tracking, activity log, API key vault commands, provisioner commands, dashboard, and templates.

## Key Insights
- These handlers are simpler than core (Phase 8); mostly CRUD with some business logic
- Approval resolution triggers side effects (agent hire on approve)
- Cost reconciliation is a periodic batch operation
- Dashboard query aggregates data from multiple tables

## Architecture
```
apps/backend/src/application/
├── commands/
│   ├── goal/
│   │   ├── create-goal-command.ts + handler
│   │   └── update-goal-command.ts + handler
│   ├── project/
│   │   ├── create-project-command.ts + handler
│   │   ├── update-project-command.ts + handler
│   │   └── create-workspace-command.ts + handler
│   ├── approval/
│   │   ├── create-approval-command.ts + handler
│   │   ├── approve-command.ts + handler
│   │   ├── reject-command.ts + handler
│   │   └── request-revision-command.ts + handler
│   ├── cost/
│   │   ├── record-cost-event-command.ts + handler
│   │   └── reconcile-budgets-command.ts + handler
│   ├── activity/
│   │   └── log-activity-command.ts + handler
│   ├── api-key-vault/
│   │   ├── store-api-key-command.ts + handler
│   │   ├── validate-api-key-command.ts + handler
│   │   └── revoke-api-key-command.ts + handler
│   └── provisioner/
│       ├── ensure-vm-command.ts + handler
│       ├── hibernate-vm-command.ts + handler
│       └── destroy-vm-command.ts + handler
├── queries/
│   ├── goal/
│   │   ├── list-goals-query.ts + handler
│   │   └── get-goal-query.ts + handler
│   ├── project/
│   │   ├── list-projects-query.ts + handler
│   │   └── get-project-query.ts + handler
│   ├── approval/
│   │   ├── list-approvals-query.ts + handler
│   │   └── get-approval-query.ts + handler
│   ├── cost/
│   │   └── get-cost-summary-query.ts + handler
│   ├── activity/
│   │   └── list-activity-query.ts + handler
│   ├── dashboard/
│   │   └── get-dashboard-summary-query.ts + handler
│   └── template/
│       ├── list-templates-query.ts + handler
│       └── get-template-query.ts + handler
└── index.ts                      # Append to CommandHandlers, QueryHandlers arrays
```

## Related Code Files

### Files to Create
~50 files (command/query pairs).

## Implementation Steps

1. **Goal commands/queries**
   - `CreateGoal`: companyId, title, description, level, parentId
   - `UpdateGoal`: partial update, validate hierarchy (no circular refs)
   - `ListGoals`: hierarchical by companyId
   - `GetGoal`: by id + company scope

2. **Project commands/queries**
   - `CreateProject`: companyId, name, description, goalId
   - `UpdateProject`: partial update, soft-archive via archivedAt
   - `CreateWorkspace`: projectId, repoUrl, repoRef, cwd, branch
   - `ListProjects`: by companyId (exclude archived)
   - `GetProject`: by id, include workspaces

3. **Approval commands/queries**
   - `CreateApproval`: type (hire_agent/approve_strategy), payload (jsonb), requestedByAgentId
   - `Approve`: set status='approved', resolvedByUserId, publish ApprovalResolvedEvent
   - `Reject`: set status='rejected', resolvedByUserId, publish ApprovalResolvedEvent
   - `RequestRevision`: set status='revision_requested', add approvalComment
   - `ListApprovals`: by companyId, filter by status
   - `GetApproval`: by id, include comments

4. **Cost commands/queries**
   - `RecordCostEvent`: Insert costEvent row, update agent.spentMonthlyCents (atomic increment), update company.spentMonthlyCents
   - `ReconcileBudgets`: Batch recalculate all agent/company spending from costEvents for current month. Run nightly.
   - `GetCostSummary`: Aggregate by date range, per-agent breakdown, per-company totals. Response matches blueprint 18 dashboard.costs shape.

5. **Activity commands/queries**
   - `LogActivity`: Insert into activityLog. Called by interceptor and handlers.
   - `ListActivity`: By companyId, paginated, optional entityType/actorType filter

6. **API Key Vault commands**
   - `StoreApiKey`: Delegate to IApiKeyVaultService.store()
   - `ValidateApiKey`: Delegate to IApiKeyVaultService.validate()
   - `RevokeApiKey`: Delegate to IApiKeyVaultService.revoke()

7. **Provisioner commands**
   - `EnsureVm`: Delegate to IProvisionerService.ensureMachine(). Returns { machineId, ip }.
   - `HibernateVm`: Delegate to IProvisionerService.hibernateMachine()
   - `DestroyVm`: Delegate to IProvisionerService.destroyMachine()

8. **Dashboard query**
   - `GetDashboardSummary`: Single query handler aggregating:
     - Agent counts by status
     - Issue counts by status
     - Cost summary for current month
     - Run statistics for today
     - Recent activity (last 20 entries)
   - Response shape per blueprint 18 dashboard/summary

9. **Template queries**
   - `ListTemplates`: All public templates, optional category filter
   - `GetTemplate`: By slug

10. **Update `index.ts`** to include all new handlers in export arrays

## Todo List
- [ ] Goal commands (2) + queries (2)
- [ ] Project commands (3) + queries (2)
- [ ] Approval commands (4) + queries (2)
- [ ] Cost commands (2) + queries (1)
- [ ] Activity commands (1) + queries (1)
- [ ] API Key Vault commands (3)
- [ ] Provisioner commands (3)
- [ ] Dashboard query (1)
- [ ] Template queries (2)
- [ ] Update handler export arrays
- [ ] typecheck passes

## Success Criteria
- All supporting handlers implemented
- Approval flow: create -> approve/reject -> event published
- Cost recording atomically updates agent + company spending
- Dashboard summary returns complete aggregated data
- `turbo typecheck` passes

## Risk Assessment
- **Dashboard query performance:** May need optimized SQL or caching for large datasets
- **Cost reconciliation race:** Use transaction for budget recalculation
- **Approval side effects:** Keep in event handler (Phase 10), not in command handler

## Security Considerations
- API key vault commands restricted to company owners/admins
- Provisioner commands are internal-only (called by heartbeat handler, not exposed via API)
- Activity log is append-only, never modified

## Next Steps
- Phase 10: Event handlers react to approval resolution, budget exceeded, etc.
