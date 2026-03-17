# Phase 5: Infrastructure - Repositories

## Context Links
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md) (repositories/ section)
- [15 - Database Design](../../docs/blueprint/04-data-and-api/15-database-design.md) (query patterns)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 3h
- **Description:** Implement all 18 repository classes using Drizzle ORM, including BaseRepository with generic CRUD, atomic checkout query, org tree queries.

## Key Insights
- Every query must filter by `companyId` for multi-tenant isolation
- `atomicCheckout` uses `UPDATE ... WHERE checkout_run_id IS NULL` for atomic lock
- Org tree query uses recursive CTE for `reportsTo` chain
- BaseRepository wraps Drizzle's query builder with common patterns
- Inject Drizzle client via `@Inject('DRIZZLE')` token

## Requirements
### Functional
- All 18 repository implementations matching domain contracts
- Multi-tenant scoping on every query
- Atomic checkout with 0-row check for conflict detection
- Org tree recursive query
- Pagination support (cursor-based preferred)

### Non-functional
- Type-safe queries (Drizzle inferred types)
- Query logging in dev mode

## Architecture
```
apps/backend/src/infrastructure/repositories/
├── base-repository.ts
├── company-repository.ts
├── agent-repository.ts
├── issue-repository.ts
├── heartbeat-run-repository.ts
├── heartbeat-run-event-repository.ts
├── goal-repository.ts
├── project-repository.ts
├── approval-repository.ts
├── cost-event-repository.ts
├── activity-repository.ts
├── company-api-key-repository.ts
├── company-vm-repository.ts
├── agent-wakeup-repository.ts
├── agent-task-session-repository.ts
├── agent-config-revision-repository.ts
├── template-repository.ts
└── index.ts
```

## Related Code Files

### Files to Create
All 18 files above + barrel index.

### Dependencies
- `apps/backend/src/domain/repositories/` (contracts from Phase 4)
- `apps/backend/src/infrastructure/persistence/schemas/` (Drizzle schemas from Phase 3)

## Implementation Steps

1. **Create `base-repository.ts`**
   ```typescript
   @Injectable()
   export class BaseRepository<TTable, TInsert, TSelect> {
     constructor(
       @Inject('DRIZZLE') protected readonly db: DrizzleClient,
       protected readonly table: TTable,
     ) {}

     async findById(id: string): Promise<TSelect | null> {
       const [row] = await this.db.select().from(this.table).where(eq(this.table.id, id)).limit(1);
       return row ?? null;
     }

     async findByIdAndCompany(id: string, companyId: string): Promise<TSelect | null> {
       const [row] = await this.db.select().from(this.table)
         .where(and(eq(this.table.id, id), eq(this.table.companyId, companyId)))
         .limit(1);
       return row ?? null;
     }

     async create(data: TInsert): Promise<TSelect> {
       const [row] = await this.db.insert(this.table).values(data).returning();
       return row;
     }

     async update(id: string, data: Partial<TInsert>): Promise<TSelect> {
       const [row] = await this.db.update(this.table)
         .set({ ...data, updatedAt: new Date() })
         .where(eq(this.table.id, id))
         .returning();
       return row;
     }

     async delete(id: string): Promise<void> {
       await this.db.delete(this.table).where(eq(this.table.id, id));
     }
   }
   ```

2. **Create `issue-repository.ts`** (most critical)
   - Extends BaseRepository
   - `atomicCheckout(issueId, agentId, runId)`:
     ```typescript
     const result = await this.db.update(issues)
       .set({ checkoutRunId: runId, assigneeAgentId: agentId, status: 'in_progress', updatedAt: new Date() })
       .where(and(eq(issues.id, issueId), isNull(issues.checkoutRunId)))
       .returning();
     return result.length > 0;
     ```
   - `releaseCheckout(issueId)`: set checkoutRunId = null
   - `getNextIdentifier(companyId, prefix)`: increment company counter + format
   - `findByCompany(companyId, filter?)`: support status, assignee, priority filters
   - `searchByCompany(companyId, query)`: ILIKE on title/description

3. **Create `agent-repository.ts`**
   - `getOrgTree(companyId)`: Recursive CTE
     ```sql
     WITH RECURSIVE org AS (
       SELECT * FROM agents WHERE company_id = ? AND reports_to IS NULL
       UNION ALL
       SELECT a.* FROM agents a JOIN org o ON a.reports_to = o.id
     )
     SELECT * FROM org;
     ```
   - `findActiveByCompany(companyId)`: status IN ('active', 'idle', 'running')
   - `updateSpentCents(id, additionalCents)`: atomic increment

4. **Create `heartbeat-run-repository.ts`**
   - `findStaleRuns(cutoff)`: status = 'running' AND updatedAt < cutoff
   - `updateStatus(id, status, extras?)`: update with optional finishedAt, exitCode, usageJson

5. **Create `heartbeat-run-event-repository.ts`**
   - `createBatch(events)`: bulk insert
   - `findByRun(runId, afterSeq?, limit?)`: ordered by seq, cursor-based pagination

6. **Create `agent-wakeup-repository.ts`**
   - `enqueue(data)`: insert with status 'queued'
   - `claimNext(agentId)`: `UPDATE ... SET status = 'claimed', claimed_at = now() WHERE agent_id = ? AND status = 'queued' ORDER BY created_at LIMIT 1 RETURNING *`
   - `coalesce(agentId)`: mark duplicate queued requests as 'coalesced'

7. **Create `company-api-key-repository.ts`**
   - `findByCompanyAndProvider(companyId, provider)`
   - `upsert(data)`: ON CONFLICT (company_id, provider) DO UPDATE

8. **Create `company-vm-repository.ts`**
   - `upsertStatus(companyId, status, machineId?)`: ON CONFLICT (company_id) DO UPDATE

9. **Create remaining repositories** (simpler CRUD + findByCompany patterns):
   - `company-repository.ts`, `goal-repository.ts`, `project-repository.ts`
   - `approval-repository.ts`, `cost-event-repository.ts`, `activity-repository.ts`
   - `agent-task-session-repository.ts`, `agent-config-revision-repository.ts`
   - `template-repository.ts`

10. **Create `index.ts`** barrel exporting all repositories

## Todo List
- [ ] BaseRepository with generic CRUD
- [ ] IssueRepository with atomic checkout
- [ ] AgentRepository with org tree CTE
- [ ] HeartbeatRunRepository with stale run finder
- [ ] HeartbeatRunEventRepository with batch insert
- [ ] AgentWakeupRepository with claim/coalesce
- [ ] CompanyApiKeyRepository with upsert
- [ ] CompanyVmRepository with upsert
- [ ] Remaining 9 repositories
- [ ] Barrel exports
- [ ] typecheck passes

## Success Criteria
- All 18 repositories implement their domain contracts
- `atomicCheckout` correctly returns false on conflict
- Org tree query returns correct hierarchy
- All queries filter by companyId where applicable
- `turbo typecheck` passes

## Risk Assessment
- **Drizzle raw SQL:** Recursive CTE may require `db.execute(sql\`...\`)` rather than query builder
- **Race conditions:** atomicCheckout relies on single UPDATE; tested in Phase 18 integration tests
- **N+1 queries:** Use Drizzle's `with` (relations) for join queries where needed

## Security Considerations
- Every tenant-scoped query MUST include companyId filter
- Never expose raw Drizzle errors to API consumers
- Parameterized queries only (Drizzle handles this)

## Next Steps
- Phase 6: Auth layer (guards use repositories for user/agent lookup)
- Phase 7: Services use repositories for business logic
