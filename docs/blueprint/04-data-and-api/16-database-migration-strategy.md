# 16 — Database Migration Strategy

Drizzle ORM migrations, partitioning, archival, and index strategy for the AI Company Platform.

## Drizzle Migration Workflow

Migrations live in `apps/backend/src/infrastructure/persistence/migrations/`.

### Generate a Migration

After modifying a Drizzle table schema, generate a migration automatically:

```bash
# From apps/backend/
pnpm drizzle-kit generate
```

Drizzle Kit diffs the current schema against the previous snapshot and writes a SQL migration file.

### Run Migrations

```bash
# Apply all pending migrations
pnpm drizzle-kit migrate

# Shortcut via turbo
turbo db:migrate --filter=@your-product/backend
```

In production, migrations run automatically on API startup via `drizzle-orm/migrator`.

### Rollback / Revert

```bash
# Drop the database and re-apply all migrations (dev only)
pnpm drizzle-kit push --force-reset

# For production, write a new migration that reverts the change
pnpm drizzle-kit generate
```

Drizzle Kit does not have a built-in revert command — write reverse migrations manually for production.

### Naming Convention

```
{timestamp}-{description}.ts

Examples:
  1700000000000-initial-schema.ts
  1700001000000-add-agent-config-revision.ts
  1700002000000-add-heartbeat-run-events-partitioning.ts
  1700003000000-add-billing-account.ts
```

The timestamp is milliseconds since epoch (`Date.now()`). Descriptions use kebab-case, concise, describes what changed.

## Adding a New Table

Standard process for adding any new domain entity:

**Step 1: Create domain entity interface**
```typescript
// domain/entities/MyEntity.ts
export interface MyEntity {
  id: string;
  companyId: string;
  // ... fields
  createdAt: Date;
  updatedAt: Date;
}
```

**Step 2: Create Drizzle table schema**
```typescript
// infrastructure/persistence/schemas/my-entities.ts
import { pgTable, uuid, timestamp, index } from 'drizzle-orm/pg-core';

export const myEntities = pgTable('my_entities', {
  id: uuid('id').primaryKey().defaultRandom(),
  companyId: uuid('company_id').notNull(),
  // ... columns
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
}, (table) => [
  index('idx_my_entities_company_created').on(table.companyId, table.createdAt),
]);
```

**Step 3: Generate and run migration**
```bash
pnpm drizzle-kit generate
pnpm drizzle-kit migrate
```

**Step 4: Create repository interface**
```typescript
// domain/repositories/IMyEntityRepository.ts
export interface IMyEntityRepository extends IBaseRepository<MyEntity> {
  findByCompany(companyId: string): Promise<MyEntity[]>;
}
```

**Step 5: Create repository implementation**
```typescript
// infrastructure/repositories/MyEntityRepository.ts
@Injectable()
export class MyEntityRepository implements IMyEntityRepository {
  constructor(
    @Inject('DRIZZLE') private readonly db: DrizzleClient,
  ) {}
  // ... implementations using db.select().from(myEntities).where(...)
}
```

**Step 6: Register in SharedModule**
```typescript
// module/shared.module.ts
DrizzleModule,
{ provide: 'IMyEntityRepository', useClass: MyEntityRepository },
```

## Partitioning Strategy for heartbeat_run_events

The `heartbeat_run_events` table is the highest-volume table (potentially millions of rows/month). It is partitioned **monthly by `created_at`** using PostgreSQL range partitioning.

### Why Partition?
- Old run events are rarely queried (dashboard shows recent runs)
- Monthly partitions allow fast archival of old data by dropping/detaching a partition
- Query performance stays constant regardless of total row count

### Partition Setup (in migration)

```sql
-- Create partitioned parent table
CREATE TABLE heartbeat_run_events (
  id UUID NOT NULL,
  company_id UUID NOT NULL,
  run_id UUID NOT NULL,
  event_type TEXT NOT NULL,
  payload JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create first partition manually (bootstrap)
CREATE TABLE heartbeat_run_events_2024_01
  PARTITION OF heartbeat_run_events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### Partition Auto-Creation (Monthly Cron)

`PartitionManagerWorker` runs on the first day of each month to pre-create the next month's partition:

```typescript
// infrastructure/workers/PartitionManagerWorker.ts
@Injectable()
export class PartitionManagerWorker {
  @Cron('0 0 1 * *') // First day of every month at 00:00
  async createNextMonthPartition() {
    const next = startOfNextMonth();
    const name = `heartbeat_run_events_${format(next, 'yyyy_MM')}`;
    await this.dataSource.query(`
      CREATE TABLE IF NOT EXISTS ${name}
        PARTITION OF heartbeat_run_events
        FOR VALUES FROM ('${format(next, 'yyyy-MM-dd')}')
                     TO ('${format(addMonths(next, 1), 'yyyy-MM-dd')}')
    `);
  }
}
```

## S3 Archival for Old Partitions

After 3 months, old partitions are detached and archived to S3:

1. **Detach partition** from parent table (queries stop hitting it)
2. **Export to Parquet/CSV** and upload to S3 (`s3://your-product-archive/run-events/2024-01/`)
3. **Drop partition** from PostgreSQL
4. Archive job runs monthly, managed by `PartitionManagerWorker`

This keeps the live database small while preserving data for audit/analytics.

## Budget Reconciliation Job

**Nightly recalculation** ensures `agents.spent_monthly_cents` and `companies.total_spent_cents` are accurate even if real-time increments drifted.

```typescript
// application/commands/cost/ReconcileBudgetsHandler.ts
// Runs nightly at 02:00 via scheduler
async execute() {
  // For each company, SUM costEvents for current month
  // Update agent.spentMonthlyCents + company.totalSpentCents
  // Emit BudgetExceededEvent for any agents now over budget
}
```

Driven by `ReconcileBudgetsCommand` dispatched by the scheduler. Source of truth is `cost_events` table — the reconciliation job re-derives aggregates from raw events.

## Monthly Budget Reset

On the first day of each month, the scheduler dispatches a reset job:

```typescript
// Resets agents.spent_monthly_cents to 0
// Resets companies.total_spent_monthly_cents to 0
// Re-activates agents that were auto-paused due to budget (if new month budget applies)
```

This is a separate migration-safe operation — the reset writes zeros into the aggregate columns, not a schema change.

## Index Strategy

These 9 indexes are critical for query performance. All must exist from the first migration.

| # | Table | Index | Purpose |
|---|-------|-------|---------|
| 1 | `agents` | `(company_id, status)` | List agents by company + status filter |
| 2 | `issues` | `(company_id, status, assignee_id)` | Kanban board, agent inbox |
| 3 | `issues` | `(checkout_run_id)` WHERE `checkout_run_id IS NOT NULL` | Atomic checkout conflict detection |
| 4 | `heartbeat_runs` | `(company_id, started_at DESC)` | Dashboard recent runs |
| 5 | `heartbeat_runs` | `(agent_id, status, started_at DESC)` | Agent detail runs tab |
| 6 | `heartbeat_run_events` | `(run_id, created_at)` | Fetch events for a specific run |
| 7 | `cost_events` | `(company_id, created_at)` WHERE month | Monthly cost aggregation |
| 8 | `activity_log` | `(company_id, created_at DESC)` | Activity feed |
| 9 | `agent_wakeup_requests` | `(agent_id, status)` WHERE `status = 'pending'` | Scheduler wakeup queue deduplication |

Add these indexes explicitly in the Drizzle schema definitions — compound indexes on partitioned tables should also be verified in the generated migration SQL.

## Neon Connection Pooling (PgBouncer)

The API server connects to Neon via **PgBouncer** (transaction-mode pooling):

```
DATABASE_URL=postgres://user:pass@ep-xxx.pooler.neon.tech:5432/neondb?sslmode=require
```

**Important Drizzle/pg settings for PgBouncer transaction mode:**
- `max`: 10 connections per API instance (not the default 100)
- Always use migrations (never use `drizzle-kit push` in production)
- Do **not** use advisory locks from the pooled connection — use a direct (non-pooled) connection for the scheduler's pg advisory lock

```typescript
// database.module.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schemas';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: { rejectUnauthorized: false },
  max: 10,              // PgBouncer transaction mode
  idleTimeoutMillis: 30000,
});

export const db = drizzle(pool, { schema });
```

For the scheduler's advisory lock, use a **separate direct connection** (non-pooled Neon URL) to avoid PgBouncer interfering with session-level advisory lock semantics.
