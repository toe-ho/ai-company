# Phase 4: Domain Layer

## Context Links
- [11 - Backend Architecture](../../docs/blueprint/03-architecture/11-backend-architecture.md)
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md) (domain/ directory)
- [21 - Error Handling](../../docs/blueprint/05-operations/21-error-handling-patterns.md)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 2h
- **Description:** Pure TypeScript domain layer. Zero framework imports. Entity interfaces, repository contracts, enums, domain exceptions, core interfaces.

## Key Insights
- Domain layer has ZERO NestJS or Drizzle imports. Pure TypeScript only.
- Entity interfaces here are backend-specific (may include methods); shared package has data-only interfaces
- Repository contracts define the API that infrastructure layer implements
- `IBaseRepository` provides generic CRUD contract; specific repos extend with domain methods
- Domain exceptions are plain Error subclasses with typed fields

## Requirements
### Functional
- 20+ entity interfaces
- 15+ repository contracts
- 10 enums (re-export from shared)
- 4 domain exceptions
- 5 core interfaces (IActor, IExecutionRequest, etc.)

### Non-functional
- Zero imports from NestJS, Drizzle, or any framework
- Only imports from `@ai-company/shared` allowed

## Architecture
```
apps/backend/src/domain/
├── entities/
│   ├── index.ts
│   └── (re-export from @ai-company/shared or extend with domain methods)
├── repositories/
│   ├── index.ts
│   ├── base-repository.interface.ts
│   ├── company-repository.interface.ts
│   ├── agent-repository.interface.ts
│   ├── issue-repository.interface.ts
│   ├── heartbeat-run-repository.interface.ts
│   ├── heartbeat-run-event-repository.interface.ts
│   ├── goal-repository.interface.ts
│   ├── project-repository.interface.ts
│   ├── approval-repository.interface.ts
│   ├── cost-event-repository.interface.ts
│   ├── activity-repository.interface.ts
│   ├── company-api-key-repository.interface.ts
│   ├── company-vm-repository.interface.ts
│   ├── agent-wakeup-repository.interface.ts
│   ├── agent-task-session-repository.interface.ts
│   ├── agent-config-revision-repository.interface.ts
│   └── template-repository.interface.ts
├── enums/
│   └── index.ts                 # Re-export from @ai-company/shared
├── interfaces/
│   └── index.ts                 # Re-export from @ai-company/shared
└── exceptions/
    ├── index.ts
    ├── issue-already-checked-out.exception.ts
    ├── agent-over-budget.exception.ts
    ├── missing-api-key.exception.ts
    └── vm-boot-failed.exception.ts
```

## Related Code Files

### Files to Create
All files above (~30 files).

## Implementation Steps

1. **Create `IBaseRepository` interface**
   ```typescript
   export interface IBaseRepository<T> {
     findById(id: string): Promise<T | null>;
     findByIdAndCompany(id: string, companyId: string): Promise<T | null>;
     findMany(filter: Record<string, unknown>): Promise<T[]>;
     create(data: Partial<T>): Promise<T>;
     update(id: string, data: Partial<T>): Promise<T>;
     delete(id: string): Promise<void>;
   }
   ```

2. **Create repository contracts** - each extends IBaseRepository with domain-specific methods:
   - `ICompanyRepository`: `findByOwnerId(userId)`, `incrementIssueCounter(id)`
   - `IAgentRepository`: `findByCompany(companyId, filter?)`, `getOrgTree(companyId)`, `findActiveByCompany(companyId)`, `updateSpentCents(id, cents)`
   - `IIssueRepository`: `findByCompany(companyId, filter?)`, `atomicCheckout(issueId, agentId, runId): Promise<boolean>`, `releaseCheckout(issueId)`, `getNextIdentifier(companyId, prefix)`, `searchByCompany(companyId, query)`
   - `IHeartbeatRunRepository`: `findByAgent(agentId, filter?)`, `findRunning(companyId?)`, `findStaleRuns(cutoff: Date)`, `updateStatus(id, status, extras?)`
   - `IHeartbeatRunEventRepository`: `createBatch(events[])`, `findByRun(runId, afterSeq?, limit?)`
   - `IGoalRepository`: `findByCompany(companyId)`, `findHierarchy(companyId)`
   - `IProjectRepository`: `findByCompany(companyId)`, `findWithWorkspaces(id)`
   - `IApprovalRepository`: `findByCompany(companyId, filter?)`, `updateStatus(id, status, resolvedBy)`
   - `ICostEventRepository`: `findByCompany(companyId, dateRange?)`, `sumByAgent(companyId, month)`, `sumByCompany(companyId, month)`
   - `IActivityRepository`: `findByCompany(companyId, filter?)`, `createEntry(data)`
   - `ICompanyApiKeyRepository`: `findByCompanyAndProvider(companyId, provider)`, `upsert(data)`
   - `ICompanyVmRepository`: `findByCompany(companyId)`, `upsertStatus(companyId, status, machineId?)`
   - `IAgentWakeupRepository`: `enqueue(data)`, `claimNext(agentId)`, `coalesce(agentId)`, `markCompleted(id)`
   - `IAgentTaskSessionRepository`: `findByAgentAndTask(agentId, taskId)`, `upsert(data)`
   - `IAgentConfigRevisionRepository`: `findByAgent(agentId)`, `getLatest(agentId)`, `createRevision(data)`
   - `ITemplateRepository`: `findAll(filter?)`, `findBySlug(slug)`

3. **Create domain exceptions**
   ```typescript
   // issue-already-checked-out.exception.ts
   export class IssueAlreadyCheckedOutException extends Error {
     constructor(
       public readonly issueId: string,
       public readonly checkedOutByRunId: string,
     ) {
       super(`Issue ${issueId} already checked out by run ${checkedOutByRunId}`);
     }
     get details() { return { issueId: this.issueId, checkedOutByRunId: this.checkedOutByRunId }; }
   }
   ```
   - `AgentOverBudgetException(agentId, budgetCents, spentCents)`
   - `MissingApiKeyException(provider)`
   - `VmBootFailedException(vmId, reason)`

4. **Create entities barrel** - re-export from `@ai-company/shared` entities
5. **Create enums barrel** - re-export from `@ai-company/shared` enums
6. **Create interfaces barrel** - re-export from `@ai-company/shared` interfaces

## Todo List
- [ ] IBaseRepository interface
- [ ] 16 repository contracts
- [ ] 4 domain exceptions
- [ ] Entity re-exports
- [ ] Enum re-exports
- [ ] Interface re-exports
- [ ] Barrel index files
- [ ] typecheck passes

## Success Criteria
- All repository contracts defined with domain-specific methods
- Zero framework imports in entire `domain/` directory
- All exceptions have typed fields
- `turbo typecheck` passes

## Risk Assessment
- **Interface bloat:** Keep IBaseRepository minimal; add methods to specific repos only when needed
- **Shared vs Domain entities:** If they diverge, backend domain can extend shared interfaces

## Security Considerations
- No secrets or infrastructure concerns in domain layer
- Exception messages safe to expose to API consumers

## Next Steps
- Phase 5: Infrastructure repositories implement these contracts
