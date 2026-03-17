# 24 — Deployment & Infrastructure

## Cloud Infrastructure Map

```
┌──────────────────────────────────────────────────────┐
│  Cloud Infrastructure                                  │
│                                                        │
│  CONTROL PLANE                                        │
│  ┌──────────────┐ ┌──────────────┐ ┌───────────────┐ │
│  │ API Server    │ │ PostgreSQL   │ │ Redis         │ │
│  │ (Railway or   │ │ (Neon)       │ │ (Upstash)     │ │
│  │  Fly.io)      │ │ Multi-tenant │ │ Pub/sub only  │ │
│  │ $10-20/mo     │ │ $0-25/mo     │ │ $0-10/mo      │ │
│  └──────┬───────┘ └──────────────┘ └───────────────┘ │
│         │                                              │
│  EXECUTION PLANE (per-company Fly.io Machines)        │
│  ┌──────┴──────┐ ┌────────────┐ ┌────────────┐       │
│  │ Company A   │ │ Company B  │ │ Company C  │       │
│  │ 4cpu/8GB    │ │ 2cpu/4GB   │ │ 2cpu/4GB   │       │
│  │ + 10GB vol  │ │ + 5GB vol  │ │ + 5GB vol  │       │
│  │ ~$10-20/mo  │ │ ~$5-10/mo  │ │ ~$5-10/mo  │       │
│  └─────────────┘ └────────────┘ └────────────┘       │
│                                                        │
│  STORAGE                                              │
│  ┌──────────────┐                                     │
│  │ S3            │ Logs, attachments                  │
│  │ $1-5/mo       │                                    │
│  └──────────────┘                                     │
└──────────────────────────────────────────────────────┘
```

## Infrastructure Costs (Your Overhead)

| Service | Provider | Cost | Notes |
|---------|----------|------|-------|
| API Server | Railway or Fly.io | $10-20/mo | Single instance to start |
| Database | Neon (free tier → Pro) | $0-25/mo | Scales with usage |
| Redis | Upstash (free tier → Pro) | $0-10/mo | Per-request billing |
| S3 Storage | AWS S3 | $1-5/mo | Pay per GB |
| Domain + SSL | Cloudflare | $0-2/mo | Free tier works |
| Monitoring | Betterstack | $0/mo | Free tier |
| **TOTAL (your overhead)** | | **$11-62/mo** | |

Per-company Fly.io VMs are billed to you, passed through to users with margin.

## Deployment Options

### Option 1: Railway (Simplest for API Server)
```bash
# Deploy via GitHub integration
# Railway auto-detects Node.js, builds, deploys
# Set env vars in Railway dashboard
# $5/mo for hobby, $20/mo for pro
```

### Option 2: Fly.io (API Server + Company VMs on Same Platform)
```bash
fly launch
fly secrets set DATABASE_URL=... AUTH_SECRET=... FLY_API_TOKEN=...
fly deploy
```

### Option 3: Docker Anywhere
```dockerfile
FROM node:lts-slim AS build
RUN pnpm install && pnpm build

FROM node:lts-slim AS production
# Pre-install: claude, codex, opencode CLIs
COPY --from=build /app/dist ./
EXPOSE 3100
CMD ["node", "server/dist/index.js"]
```

## CI/CD (GitHub Actions)

### PR Verify
```yaml
on: pull_request
steps:
  - pnpm install
  - pnpm typecheck
  - pnpm test:run
  - pnpm build
```

### Deploy to Production
```yaml
on:
  push:
    branches: [main]
steps:
  - pnpm install && pnpm build
  - Deploy to Railway/Fly.io
  - Run migrations
  - Health check
```

### E2E Tests
```yaml
on: workflow_dispatch
steps:
  - Install Playwright
  - Start server
  - Run E2E suite
  - Upload report
```

## Fly.io Machine Management

### Company VM Provisioning
```typescript
// When first heartbeat triggers for a company:
const machine = await fly.machines.create({
  app: FLY_APP_NAME,
  config: {
    image: "your-org/agent-executor:latest",
    guest: { cpus: 2, memory_mb: 4096 },
    mounts: [{ volume: volumeId, path: "/workspace" }],
    env: {
      CONTROL_PLANE_URL: "https://your-api.com",
      COMPANY_ID: companyId,
    }
  }
});
```

### Auto-Hibernate
```typescript
// After heartbeat completes + idle timeout:
await fly.machines.stop(machineId);
// VM sleeps, $0 cost
// Persistent volume preserved

// On next heartbeat:
await fly.machines.start(machineId);
// ~3 second wake, workspace intact
```

### Agent Executor Docker Image
```dockerfile
FROM node:lts-slim
RUN npm install -g claude codex opencode
COPY agent-executor/ ./
EXPOSE 8080
CMD ["node", "index.js"]
```

Pre-installed agent CLIs. Listens for execution requests from the control plane.

## Scheduler in Multi-Replica Deployments

The heartbeat scheduler runs inside `apps/backend/` via `@nestjs/schedule`. When deploying multiple backend replicas:

- **PostgreSQL advisory lock** prevents duplicate tick execution — only one replica runs the scheduler tick at a time
- If the lock holder crashes, the lock is automatically released and another replica picks up
- No configuration needed — the lock is acquired per-tick (not per-startup), so failover is immediate
- Safe to scale to N replicas without scheduler conflicts

## Database Hosting

| Option | Cost | Notes |
|--------|------|-------|
| **Neon** (free → $19/mo) | Serverless, auto-scale | Default recommendation |
| **Supabase** (free → $25/mo) | Postgres + extras | If you want auth/storage too |

## Monitoring & Observability

### Built-in
- **Activity log** — Every mutation recorded
- **Cost dashboard** — Per-agent, per-task spending
- **Run history** — Full execution logs per heartbeat
- **Health endpoint** — `GET /api/health`
- **VM status** — Per-company Fly.io machine state

### External (Recommended)
- **Betterstack** (free) — Uptime monitoring, log aggregation
- **Sentry** (free tier) — Error tracking
- **Fly.io metrics** — Built-in VM monitoring

## Scaling Path

### Phase 1 (0-100 companies)
- Single API server instance
- Single Neon database
- Upstash Redis free tier
- Works fine

### Phase 2 (100-1,000 companies)
- 2-3 API server instances (load balanced)
- Neon Pro ($19/mo, connection pooling)
- Upstash Pro ($10/mo)
- S3 for log storage

### Phase 3 (1,000+ companies)
- Horizontal API scaling (4+ instances)
- Database read replicas
- Redis cluster
- CDN for static assets
- Consider dedicated Fly.io organization for VMs

## Backup & Recovery

- **Database:** Neon/Supabase handle backups automatically
- **VM volumes:** Fly.io snapshots (manual or scripted)
- **S3:** Versioning enabled for file storage
