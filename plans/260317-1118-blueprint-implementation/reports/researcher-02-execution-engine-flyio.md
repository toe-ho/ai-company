# Execution Engine & Fly.io Integration Research

**Date:** 2026-03-17
**Focus:** AI agent execution platform patterns, Fly.io Machines, SSE streaming, sandboxing, real-time dashboards

---

## 1. Fly.io Machines API: VM Lifecycle Management

### Operational Pattern
- **REST API** over HTTPS with Bearer token auth (Fly.io API key)
- **Lifecycle:** Create → (optional skip_launch) → Start → Stop → Destroy
- **Speed advantage:** Subsecond start/stop (boot times measured in milliseconds)
- **Per-company isolation:** Each company gets dedicated Fly app in their region

### Implementation Details
- No special tooling required—curl/fetch works
- JSON request/response format
- Machine states tracked server-side; host receives direct start/stop messages
- Scaling: Create multiple machines per company for parallel agent execution
- Region placement: Specify region in create request for data residency compliance

### Cost Optimization
- Pay only when machines are running (stop when idle between heartbeats)
- Billing per machine-hours
- No ongoing cost during stopped state

**Reference:** [Fly Machines API Docs](https://docs.machines.dev/), [Machine Lifecycle Overview](https://fly.io/docs/machines/machine-states/)

---

## 2. SSE Streaming for Agent Results

### Why SSE > WebSockets for Agent Execution
- **One-way streaming** (perfect for control plane → client result streaming)
- **Simpler protocol:** HTTP-based, no connection state overhead
- **Native browser support:** Built-in reconnection with message IDs
- **Lower latency:** No handshake like WebSocket

### NestJS Implementation Pattern
```typescript
@Sse('agent-results/:companyId/:agentId')
streamAgentResults(@Param('companyId') cid: string) {
  return fromEvent(eventEmitter, `result:${cid}`);
}
```
- Use `@Sse()` decorator on controller method
- Return Observable stream (RxJS `fromEvent`)
- Client receives `event: data\n\n` formatted messages
- Browser auto-reconnects on disconnect

### Practical Pattern
1. Client opens SSE connection: `GET /agent-results/{companyId}/{agentId}`
2. Control plane dispatches heartbeat to Fly machine
3. Machine streams logs/progress back to control plane (HTTP callback or persistent connection)
4. Control plane emits to SSE subscribers in real-time
5. Connection persists until agent completes

**Reference:** [NestJS SSE Documentation](https://docs.nestjs.com/techniques/server-sent-events), [Real-Time Log Streaming Guide](https://dev.to/manojspace/real-time-log-streaming-with-nodejs-and-react-using-server-sent-events-sse-48pk)

---

## 3. Agent Sandboxing: Isolation & Cost Control

### Isolation Architecture
**Pattern: Isolate the Agent** (control plane holds all secrets)
- Agent runs in sandbox with zero credentials
- Sandbox receives only: SESSION_TOKEN, CONTROL_PLANE_URL, SESSION_ID
- No direct AWS/database access from sandbox
- File sync service watches /workspace, syncs to S3 (secrets never stored in files)

### Technology Options
| Tech | Strength | Trade-off |
|------|----------|-----------|
| **MicroVMs** (Firecracker) | Strongest isolation, dedicated kernel | Higher latency |
| **gVisor** | Syscall interception, medium isolation | CPU overhead |
| **Hardened Containers** | Fast startup | For trusted code only |

For Fly.io: Use hardened containers with Seccomp profiles + namespace isolation.

### Cost Tracking Implementation
- **Control plane enforces caps:** Max tokens per execution, max concurrent agents
- **Billing separation:** Pay for sandbox runtime only during code execution, not idle time
- **Per-company quotas:** Environment variable `MAX_EXECUTION_TOKENS`, `COST_CAP_USD`
- **Tracking:** Log token usage + duration, aggregate in control plane

### Workspace Isolation
- Agent writes to `/workspace` only (read-write mount point)
- External syncs via S3 pre-signed URLs (injected at startup)
- File quota enforced (e.g., 5GB per execution)

**Reference:** [Browser-Use Sandbox Architecture](https://browser-use.com/posts/two-ways-to-sandbox-agents), [NVIDIA Agentic Workflow Security](https://developer.nvidia.com/blog/practical-security-guidance-for-sandboxing-agentic-workflows-and-managing-execution-risk/), [Google Cloud Agent Sandbox](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/agent-sandbox)

---

## 4. Real-Time Dashboard: Redis Pub/Sub + WebSocket

### Architecture (Scaled Setup)
```
Fly Machines (Agent Execution)
         ↓ (HTTP callback + metrics)
Control Plane (NestJS)
         ↓ (emit event)
Redis Pub/Sub (Upstash)
         ↓ (broadcast)
NestJS WebSocket Gateway (multiple instances)
         ↓
Dashboard Clients
```

### NestJS Implementation
```typescript
// Gateway setup
@WebSocketGateway({ transports: ['websocket'] })
export class MetricsGateway {
  constructor(private redis: Redis) {}

  @SubscribeMessage('metrics')
  async onMetrics(client: Socket, data: AgentMetrics) {
    // Publish to Redis for all connected gateways
    this.redis.publish('agent-metrics', JSON.stringify(data));
  }
}

// On control plane metrics event:
this.redis.publish('agent-metrics', JSON.stringify({
  companyId, agentId, tokenUsed, duration, status
}));
```

### Upstash Integration
- Serverless Redis (no infrastructure management)
- REST API fallback (if WebSocket unavailable)
- Auto-scaling: Handles spikes in metric publishing
- Cost: Pay per request (ideal for bursty dashboard updates)

### Dashboard Update Pattern
1. Control plane publishes metrics to Redis channel: `agent-metrics:{companyId}`
2. Gateway subscribers listen on channel
3. Broadcast to all connected clients for that company via WebSocket
4. Fallback to HTTP polling if WebSocket unavailable

**Reference:** [NestJS Redis Pub/Sub Scaling](https://medium.com/@bhagyarana80/scaling-nestjs-gateways-with-redis-pub-sub-across-instances-b927ac41bb31), [Upstash Real-Time Notifications](https://upstash.com/blog/realtime-notifications), [Socket.IO + Redis Pub/Sub](https://dev.to/codexam/scaling-real-time-communication-with-redis-pubsub-and-socketio-3p56)

---

## 5. Control Plane Architecture: Heartbeat Dispatch Pattern

### Two-Phase Execution Flow
**Phase 1 (Diastole - Expansion):**
- Control plane sends heartbeat: `POST /execute` to Fly machine with task
- Machine executes agent (with cost/token limits)
- Machine streams progress back via callback URL or persistent connection

**Phase 2 (Systole - Contraction):**
- Machine returns results to control plane
- Control plane aggregates results, publishes to Redis
- Dashboard receives update via WebSocket

### Dispatch Logic
```typescript
// Control plane
async dispatchHeartbeat(companyId: string, agentId: string, task: Task) {
  const machine = await flyApi.getMachine(companyId, agentId);
  if (machine.state !== 'started') {
    await flyApi.startMachine(machine.id);
  }

  const result = await fetch(`http://${machine.privateIp}:3000/execute`, {
    method: 'POST',
    body: JSON.stringify({
      task,
      callbackUrl: `https://control-plane.com/results/${companyId}/${agentId}`,
      costCap: 1000, // tokens
      timeout: 300000 // 5 min
    })
  });

  // Optionally stop machine after execution to save costs
  await flyApi.stopMachine(machine.id);
}
```

### Polling vs Callback Strategy
- **Heartbeat interval:** 10-60 seconds (configurable per company)
- **Callback (preferred):** Machine posts results back to control plane webhook
- **Polling fallback:** Control plane queries machine `/status` endpoint periodically

### Governance & Control
- Query active agents before dispatch (check if agent in execution)
- Match tasks to agents based on availability + priority queue
- Mark execution as `completed` or `failed` in database
- Log all executions for audit + billing

**Reference:** [Agentic Heartbeat Pattern](https://medium.com/@marcilio.mendonca/the-agentic-heartbeat-pattern-a-new-approach-to-hierarchical-ai-agent-coordination-4e0dfd60d22d), [AI Agent Control Plane](https://www.ruh.ai/blogs/control-plane-problem-ai-agents-data-leaks-architecture-failure), [Microsoft AI Agent Orchestration](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)

---

## Key Actionable Patterns

| Component | Pattern | Why |
|-----------|---------|-----|
| **VM Lifecycle** | Stop after execution | Minimize cost (pay only during execution) |
| **Result Streaming** | SSE (not WebSocket) | Simpler for one-way streaming, auto-reconnect |
| **Isolation** | Sandbox with env vars only | Secrets never leak, cost capped |
| **Dashboard** | Redis Pub/Sub → WebSocket | Scales across multiple control plane instances |
| **Dispatch** | Heartbeat + callback | Async, fault-tolerant, easy to monitor |

---

## Integration Checklist

- [ ] Implement Fly.io Machines API client (create/start/stop/destroy)
- [ ] Set up SSE endpoint in NestJS for result streaming
- [ ] Configure Fly machine image with agent runtime + cost tracking
- [ ] Implement sandbox environment variable injection in Fly machine
- [ ] Wire Redis Pub/Sub in control plane for metrics publishing
- [ ] Build WebSocket gateway for real-time dashboard
- [ ] Implement heartbeat dispatch loop (10-60 sec interval)
- [ ] Add cost cap enforcement (token limits per execution)
- [ ] Create Upstash Redis instance (or self-hosted Redis)
- [ ] Test per-company VM isolation and region placement

---

## Unresolved Questions

1. Should callback URL or polling be primary result collection? (Callback more efficient, but polling simpler for stateless design)
2. How to handle long-running agents (>15 min)? Heartbeat interval? Stop & resume?
3. Where to store execution history—PostgreSQL or time-series DB (InfluxDB, TimescaleDB)?
4. Upstash vs self-hosted Redis trade-offs for this scale?
