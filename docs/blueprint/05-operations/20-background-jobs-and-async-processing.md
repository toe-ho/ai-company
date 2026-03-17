# 20 — Background Jobs & Async Processing

## Heartbeat Scheduler (Built-in Module)

The scheduler runs inside `apps/backend/` as `SchedulerModule` using `@nestjs/schedule`. No separate process needed.

### How It Works
```
Backend startup
    │
    ├── @nestjs/schedule registers interval job
    │
    ▼
@Interval(30000)  tick()    ← Every 30 seconds
    │
    ├── Acquire pg advisory lock (skip if another replica holds it)
    │
    ▼
Each tick (only if lock acquired):
  1. reapOrphanedRuns()      — Stale runs (>5 min) → "failed"
  2. resumeQueuedRuns()      — Resume runs queued before restart
  3. tickTimers(now)         — Check heartbeat intervals, enqueue runs
```

### Per-Agent Config
```json
{
  "runtimeConfig": {
    "heartbeatEnabled": true,
    "heartbeatIntervalSec": 300
  }
}
```

Minimum interval: 30 seconds. Agents with `heartbeatEnabled: false` only run on-demand.

## Wakeup Request Queue

Database-backed queue (not Redis/RabbitMQ):

### Lifecycle
```
queued → claimed → completed/failed
queued → coalesced (merged with duplicate)
queued → skipped (agent paused/over budget)
```

### Coalescing
Multiple wakeups for same agent within short window merge:
- First request claimed
- Subsequent requests marked `coalesced`
- `coalescedCount` tracks merges
- Prevents redundant heartbeats

### Event-Driven Wakeups
| Event | Creates Wakeup With |
|-------|-------------------|
| Task assigned | `source: "assignment", taskId` |
| Comment mention | `source: "on_demand", commentId` |
| Manual "Run" | `source: "on_demand", trigger: "manual"` |
| Approval resolved | `source: "automation", approvalId` |
| Self-hire approved | `source: "automation"` |

## VM Lifecycle Manager

### Idle Detection + Hibernate
```
Agent heartbeat completes
    │
    ├─ Start idle timer (10 min default)
    │
    ▼
No new heartbeats within 10 min?
    │
    ├─ YES → Provisioner.hibernateMachine(companyId)
    │         VM sleeps, $0 cost while sleeping
    │         Persistent volume preserved
    │
    └─ NO → Timer resets
```

### Wake on Demand
```
New heartbeat for company with hibernated VM:
    │
    ├─ Provisioner.ensureMachine(companyId)
    │   → Fly.io wakes VM (~3 seconds)
    │   → Wait for Agent Executor ready
    │
    ▼
Proceed with heartbeat execution
```

### VM States
```
stopped ──► starting ──► running ──► hibernating ──► stopped
                              │
                              └──► destroyed (company deleted)
```

## Real-time Event Distribution

### Redis Pub/Sub
```
Service mutation
    │
    ├─ PUBLISH to Redis channel: "company:{companyId}"
    │
    ▼
API Server (subscriber)
    │
    ├─ Forward to WebSocket connections for that company
    │
    ▼
Browser receives: { type: "issue.updated", payload: {...} }
```

### Event Types
```
heartbeat.run.started
heartbeat.run.event         ← Each log chunk
heartbeat.run.completed
issue.created / updated
issue.checked_out / released
agent.status_changed
agent.budget_warning
agent.budget_exceeded
approval.created / resolved
cost.alert
```

## Run Log Streaming

During execution, agent output streams in real-time:

```
Agent stdout/stderr
    │ (in Agent Executor on Fly.io VM)
    ▼
onLog() callback → INSERT heartbeatRunEvents (seq number)
    │
    ▼
Clients consume:
  UI: WebSocket push or 10-15s polling
  GET /heartbeat-runs/:id/events?afterSeq=N
```

## No External Queue System (V1)

All async processing is:
- **Database-backed:** Wakeup requests table for task queue
- **Built-in scheduler:** `@nestjs/schedule` `@Interval()` inside backend, pg advisory lock for multi-replica safety
- **Redis:** Only for pub/sub events, not job queues

Simple, zero-dependency approach. External job queues (BullMQ, etc.) can be added later if needed.

## Cost Reconciliation (Background)

Periodic job to reconcile Fly.io compute costs:
```
Every 15 minutes:
  1. Poll Fly.io billing API for machine usage
  2. Attribute seconds to company
  3. Update companyVms.computeCostCents
  4. Check company budget
  5. Alert if approaching limit
```
