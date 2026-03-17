# Phase 13: Real-time & Scheduler

## Context Links
- [20 - Background Jobs](../../docs/blueprint/05-operations/20-background-jobs-and-async-processing.md)
- [11 - Backend Architecture](../../docs/blueprint/03-architecture/11-backend-architecture.md) (scheduler section)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 2h
- **Description:** WebSocket gateway for real-time events, heartbeat scheduler with pg advisory lock, background workers.

## Key Insights
- Scheduler uses `@nestjs/schedule` `@Interval(30000)` -- NOT a separate process
- Pg advisory lock ensures only one replica runs the tick at a time
- WebSocket gateway subscribes to Redis pub/sub per company
- Clients join a room per company; events fan out to all connections in room
- Background workers: budget reconciliation (every 15 min). PartitionManagerWorker deferred to post-MVP.
<!-- Updated: Validation Session 1 - Defer partitioning to post-MVP -->

## Architecture
```
apps/backend/src/
├── presentation/gateways/
│   └── realtime-gateway.ts
├── module/
│   ├── scheduler.module.ts       # Updated from placeholder
│   └── realtime.module.ts        # Updated from placeholder
└── infrastructure/workers/
    ├── heartbeat-scheduler-service.ts
    ├── budget-reconciliation-worker.ts
    └── partition-manager-worker.ts
```

## Related Code Files

### Files to Create
- `apps/backend/src/presentation/gateways/realtime-gateway.ts`
- `apps/backend/src/infrastructure/workers/heartbeat-scheduler-service.ts`
- `apps/backend/src/infrastructure/workers/budget-reconciliation-worker.ts`
- `apps/backend/src/infrastructure/workers/partition-manager-worker.ts`

### Files to Modify
- `apps/backend/src/module/scheduler.module.ts`
- `apps/backend/src/module/realtime.module.ts`

## Implementation Steps

1. **Create `RealtimeGateway`**
   ```typescript
   @WebSocketGateway({ path: '/api/ws', cors: { origin: '*', credentials: true } })
   export class RealtimeGateway implements OnModuleInit, OnModuleDestroy {
     @WebSocketServer() server: Server;

     constructor(
       @Inject('ILiveEventsService') private liveEvents: ILiveEventsService,
     ) {}

     onModuleInit() {
       // Subscribe to Redis; forward to WebSocket rooms
     }

     @SubscribeMessage('join')
     handleJoin(client: Socket, payload: { companyId: string }) {
       client.join(`company:${payload.companyId}`);
       return { event: 'joined', data: { companyId: payload.companyId } };
     }

     @SubscribeMessage('leave')
     handleLeave(client: Socket, payload: { companyId: string }) {
       client.leave(`company:${payload.companyId}`);
     }

     // Called by Redis subscriber
     broadcastToCompany(companyId: string, event: unknown) {
       this.server.to(`company:${companyId}`).emit('event', event);
     }
   }
   ```
   - Subscribe to Redis `company:*` pattern on init
   - Forward events to WebSocket rooms
   - Handle client join/leave

2. **Create `HeartbeatSchedulerService`**
   ```typescript
   @Injectable()
   export class HeartbeatSchedulerService {
     constructor(
       @Inject('DRIZZLE') private db: DrizzleClient,
       private commandBus: CommandBus,
     ) {}

     @Interval(30000) // Every 30 seconds
     async tick() {
       // Acquire pg advisory lock
       const lockResult = await this.db.execute(
         sql`SELECT pg_try_advisory_lock(1) AS acquired`
       );
       if (!lockResult[0]?.acquired) return; // Another replica holds the lock

       try {
         // 1. Reap orphaned runs (>5 min stale)
         await this.commandBus.execute(new ReapOrphanedRunsCommand());

         // 2. Resume queued runs
         await this.resumeQueuedRuns();

         // 3. Tick heartbeat timers
         await this.tickTimers();
       } finally {
         await this.db.execute(sql`SELECT pg_advisory_unlock(1)`);
       }
     }

     private async tickTimers() {
       // Find agents with heartbeatEnabled=true where
       // lastHeartbeatAt + heartbeatIntervalSec < now()
       // For each: execute InvokeHeartbeatCommand
     }

     private async resumeQueuedRuns() {
       // Find wakeup requests with status='queued'
       // Claim and execute
     }
   }
   ```

3. **Create `BudgetReconciliationWorker`**
   ```typescript
   @Injectable()
   export class BudgetReconciliationWorker {
     @Cron('0 */15 * * * *') // Every 15 minutes
     async reconcile() {
       await this.commandBus.execute(new ReconcileBudgetsCommand());
     }
   }
   ```

4. **Create `PartitionManagerWorker`**
   - Runs monthly (Cron: `0 0 1 * *`)
   - Creates next month's partition for heartbeatRunEvents table (if using partitioning)
   - V1: No-op placeholder; partitioning added when table exceeds millions of rows

5. **Update `SchedulerModule`**
   ```typescript
   @Module({
     imports: [ScheduleModule.forRoot()],
     providers: [
       HeartbeatSchedulerService,
       BudgetReconciliationWorker,
       PartitionManagerWorker,
     ],
   })
   export class SchedulerModule {}
   ```

6. **Update `RealtimeModule`**
   ```typescript
   @Module({
     providers: [RealtimeGateway],
   })
   export class RealtimeModule {}
   ```

## Todo List
- [ ] RealtimeGateway (WebSocket + Redis bridge)
- [ ] HeartbeatSchedulerService (30s tick + pg advisory lock)
- [ ] BudgetReconciliationWorker (15min cron)
- [ ] PartitionManagerWorker (placeholder)
- [ ] Update SchedulerModule
- [ ] Update RealtimeModule
- [ ] typecheck passes

## Success Criteria
- WebSocket clients receive events in real-time per company
- Scheduler tick runs every 30s on exactly one replica
- Pg advisory lock correctly prevents duplicate execution
- Budget reconciliation runs periodically
- `turbo typecheck` passes

## Risk Assessment
- **Advisory lock with PgBouncer:** Advisory locks require direct connection (not transaction-mode pooler). Use a separate connection for scheduler.
- **WebSocket scaling:** Single server OK for V1. For multiple replicas, Redis adapter for Socket.io needed.
- **Tick duration:** If tick takes >30s, next tick overlaps. Advisory lock prevents this, but log warnings.

## Security Considerations
- WebSocket connections should validate session cookie before allowing join
- Company room membership verified (user must have access to company)
- No sensitive data in WebSocket events (IDs only, not API keys)

## Next Steps
- Phase 14: Executor app (the other end of ExecutionEngineService)
