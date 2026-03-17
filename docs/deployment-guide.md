# Deployment Guide

Complete infrastructure setup, deployment procedures, and operational runbooks for AI Company Platform v2.

---

## Cloud Infrastructure Stack

| Component | Provider | Purpose | Cost/mo |
|-----------|----------|---------|---------|
| PostgreSQL | Neon | Managed database | $5-50 |
| Redis | Upstash | Pub/sub, caching | $10-20 |
| API Server | Railway or Fly.io | Control plane | $10-50 |
| Frontend | Vercel or Fly.io | React SPA | $5-20 |
| File Storage | AWS S3 | Backups, logs | $5-10 |
| Agent VMs | Fly.io | Executor machines | $1-2 per VM |
| CI/CD | GitHub Actions | Build & test | Free (included) |
| Monitoring | Axiom + Sentry | Logs & errors | $5-20 |

**Typical monthly cost:** $50-100 for MVP

---

## Database Setup (Neon PostgreSQL)

### 1. Create Neon Account & Project

```bash
# Visit https://console.neon.tech
# Sign up with GitHub or email
# Create new project
```

### 2. Connection String

```bash
# Neon provides:
postgresql://user:password@host/database?sslmode=require
```

**Store in environment variable:**
```bash
DATABASE_URL=postgresql://user:***@ep-xyz.us-east-1.neon.tech/db?sslmode=require
```

### 3. Initialize Database

```bash
cd apps/backend

# Run migrations
pnpm db:migrate

# Seed test data
pnpm db:seed
```

**Schema includes:**
- 35+ tables (multi-tenant)
- Primary indexes on companyId, agentId, createdAt
- JSONB fields for flexible configs
- Foreign key constraints

### 4. Replication & Backup

Neon auto-manages:
- **Replication:** 3x redundancy
- **Backups:** Daily, 7-day retention
- **Failover:** Automatic on primary failure

**Manual backups:**
```bash
# Export data to S3
pg_dump $DATABASE_URL | gzip | aws s3 cp - s3://backups/db-$(date +%Y%m%d).gz
```

### 5. Connection Pooling

**PgBouncer config (built into Neon):**
- Min connections: 5
- Max connections: 20
- Idle timeout: 5 min
- Query timeout: 30s

**Override in pool manager (if needed):**
```typescript
// src/infrastructure/database/db.ts
const pool = new Pool({
  host: env.DB_HOST,
  port: 5432,
  database: env.DB_NAME,
  user: env.DB_USER,
  password: env.DB_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

---

## Redis Setup (Upstash)

### 1. Create Upstash Account

```bash
# Visit https://console.upstash.com
# Create Redis database
```

### 2. Connection Details

```bash
# Upstash provides:
redis://default:password@host:port
```

**Store in environment:**
```bash
REDIS_URL=redis://default:***@us1-xyz.upstash.io:123
```

### 3. Redis Pub/Sub Configuration

**Channels used:**
```
company:{companyId}:execution     → Execution events
company:{companyId}:issues        → Issue updates
company:{companyId}:costs         → Cost updates
company:{companyId}:approvals     → Approval notifications
```

**Pub/Sub client setup:**
```typescript
// src/modules/realtime/redis.service.ts
import { createClient } from 'redis';

const client = createClient({ url: env.REDIS_URL });
client.on('error', (err) => logger.error('Redis error', err));
await client.connect();
```

### 4. Cache Configuration

**Cache key patterns:**
```
company:{companyId}                → Company config (TTL: 10min)
agents:{companyId}                 → Agent list (TTL: 3min)
agent:{agentId}:profile            → Agent details (TTL: 5min)
costs:{companyId}:monthly          → Monthly costs (TTL: 1min)
```

**Cache invalidation:**
```typescript
// On company update
await redis.del(`company:${companyId}`);

// On agent change
await redis.del(`agents:${companyId}`);
```

---

## API Server Deployment

### Option A: Railway (Recommended for Simplicity)

#### 1. Connect GitHub

```bash
# Visit https://railway.app
# Sign up with GitHub
# Connect GitHub repo
```

#### 2. Configure Environment

```bash
# In Railway dashboard:
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
JWT_SECRET=<generate random 32+ char string>
ENCRYPTION_KEY=<generate random 32-char base64>
NODE_ENV=production
```

#### 3. Deploy

```bash
# Railway auto-deploys on push to main
# Logs available in dashboard
```

#### 4. Custom Domain

```bash
# In Railway project settings:
# Custom Domain → api.yourcompany.com
# CNAME: api.yourcompany.com → railway.app
```

### Option B: Fly.io

#### 1. Install Fly CLI

```bash
# macOS
brew install flyctl

# Linux
curl -L https://fly.io/install.sh | sh
```

#### 2. Create App

```bash
cd apps/backend

fly launch --name ai-company-api
# Fly generates fly.toml

# Configure environment
fly secrets set DATABASE_URL=postgresql://...
fly secrets set REDIS_URL=redis://...
fly secrets set JWT_SECRET=...
fly secrets set ENCRYPTION_KEY=...
```

#### 3. Deploy

```bash
fly deploy
```

#### 4. Logs & Monitoring

```bash
fly logs
fly status
fly open  # Open app in browser
```

### 3. Build & Runtime

**Dockerfile:**
```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy only package files first (cache layer)
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build
RUN pnpm build

# Start
EXPOSE 3000
CMD ["node", "apps/backend/dist/main.js"]
```

**Build command:**
```bash
pnpm build  # Compiles all packages
```

**Start command:**
```bash
node apps/backend/dist/main.js
```

### 4. Health Check

**Endpoint:** `GET /health`

```typescript
// src/presentation/controllers/health.controller.ts
@Controller('health')
export class HealthController {
  @Get()
  health() {
    return { status: 'ok', timestamp: new Date() };
  }
}
```

**Railway/Fly.io health check:**
```yaml
# fly.toml or Railway config
healthCheck:
  uri: /health
  interval: 10s
  timeout: 5s
  retries: 2
```

---

## Executor App Deployment (Fly.io Machines)

### 1. Per-Company VM Creation

**When company signs up:**
```typescript
// src/application/commands/create-company.handler.ts
async executeCreateCompany(command: CreateCompanyCommand) {
  const company = new Company(command);

  // Create Fly.io Machine
  const flyApi = new FlyApiClient(env.FLY_API_TOKEN);
  const vm = await flyApi.createMachine({
    appName: `ai-co-${company.id}`,
    region: 'sjc', // San Jose (US)
    image: 'registry.fly.io/ai-company-executor:latest',
    scale: { memory: 256, cpu: 1 },
    env: {
      COMPANY_ID: company.id,
      API_URL: env.API_URL,
      ADAPTER_TYPES: 'claude,openclaw,process',
    },
  });

  company.vmId = vm.id;
  await this.companyRepo.save(company);
}
```

### 2. Docker Build & Push

```bash
cd apps/executor

# Build image
docker build -t executor:latest .

# Tag for Fly.io
docker tag executor:latest registry.fly.io/ai-company-executor:latest

# Authenticate
fly auth docker

# Push
docker push registry.fly.io/ai-company-executor:latest
```

### 3. Fly.io Machine Lifecycle

**Hibernation:**
```bash
# VM sleeps after 5 min idle, wakes on request
# Cost: ~$0.001 per VM per hour (when awake)

# In executor health check:
if (lastRequestTime < 5min ago) {
  return { status: 'hibernating' };
}
```

**Auto-restart:**
```yaml
# fly.toml
app = "ai-company-executor"
kill_signal = "SIGINT"
kill_timeout = 5s

[processes]
app = "node apps/executor/dist/main.js"
```

**Stop/Remove VM:**
```typescript
// When company is deleted
const flyApi = new FlyApiClient(env.FLY_API_TOKEN);
await flyApi.destroyMachine(company.vmId);
```

---

## Frontend Deployment

### Option A: Vercel (Recommended for React)

#### 1. Connect GitHub

```bash
# Visit https://vercel.com
# Import project
# Vercel auto-detects Vite + React
```

#### 2. Environment Variables

```bash
# In Vercel dashboard:
VITE_API_URL=https://api.yourcompany.com
VITE_WS_URL=wss://api.yourcompany.com
```

#### 3. Deploy

```bash
# Vercel auto-deploys on push to main
# Builds with: pnpm install && pnpm build
```

#### 4. Custom Domain

```bash
# In Vercel settings:
# Add custom domain: app.yourcompany.com
# Update DNS CNAME to Vercel endpoint
```

### Option B: Fly.io

```bash
cd apps/web

fly launch --name ai-company-web
fly deploy
```

### 3. Build Configuration

**Vite config (apps/web/vite.config.ts):**
```typescript
export default defineConfig({
  plugins: [react()],
  build: {
    outDir: 'dist',
    sourcemap: false, // Disable in prod
    minify: 'terser',
    chunkSizeWarningLimit: 1000,
  },
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
});
```

**Build command:**
```bash
pnpm build  # Outputs to apps/web/dist
```

---

## File Storage (AWS S3)

### 1. Create S3 Bucket

```bash
aws s3api create-bucket \
  --bucket ai-company-backups \
  --region us-east-1
```

### 2. IAM Credentials

```bash
# Create IAM user with S3 access
# Store in environment:
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET=ai-company-backups
```

### 3. Usage

**Upload execution logs:**
```typescript
// src/infrastructure/clients/s3.client.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({ region: 'us-east-1' });

async uploadLog(companyId: string, runId: string, log: string) {
  await s3.send(new PutObjectCommand({
    Bucket: env.S3_BUCKET,
    Key: `logs/${companyId}/${runId}.log`,
    Body: log,
  }));
}
```

---

## Environment Variables

### Backend (.env.backend)

```bash
# Database
DATABASE_URL=postgresql://user:pass@host/db

# Redis
REDIS_URL=redis://user:pass@host:6379

# Auth
JWT_SECRET=your-secret-key-32-chars-min
JWT_EXPIRY=10m
ENCRYPTION_KEY=base64-encoded-32-bytes

# Better Auth
BETTER_AUTH_SECRET=random-secret

# Fly.io (for executor VMs)
FLY_API_TOKEN=fm1_your_token_here
FLY_APP_NAME=ai-company-executor

# API
API_URL=https://api.yourcompany.com
API_PORT=3000
NODE_ENV=production

# Logging
LOG_LEVEL=info
AXIOM_API_TOKEN=xxx

# Sentry (error tracking)
SENTRY_DSN=https://key@sentry.io/project

# AWS S3
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET=ai-company-backups
```

### Frontend (.env.web)

```bash
VITE_API_URL=https://api.yourcompany.com
VITE_WS_URL=wss://api.yourcompany.com
VITE_APP_NAME=AI Company Platform
VITE_SUPPORT_EMAIL=support@yourcompany.com
```

### Executor (.env.executor)

```bash
COMPANY_ID=company-uuid
API_URL=https://api.yourcompany.com
API_TOKEN=jwt-token-from-api
EXECUTOR_PORT=8080
LOG_LEVEL=info
ADAPTER_TYPES=claude,openclaw,process
```

---

## Database Migrations

### Development

```bash
cd apps/backend

# Create new migration
pnpm db:generate-migration --name add_agents_table

# Run migrations
pnpm db:migrate

# Seed test data
pnpm db:seed
```

### Production

```bash
# Before deploying new version
fly ssh console  # SSH into Railway/Fly.io container

# Run migrations
DATABASE_URL=$DATABASE_URL node dist/infrastructure/database/migrate.js

# Verify
psql $DATABASE_URL -c "SELECT table_name FROM information_schema.tables WHERE table_schema='public';"
```

**Rollback (if needed):**
```bash
# Revert to previous schema (Neon branching)
# Or restore from daily backup
```

---

## CI/CD Pipeline (GitHub Actions)

### Workflow (.github/workflows/deploy.yml)

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      - uses: actions/setup-node@v3
        with:
          node-version: 20
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: TypeCheck
        run: pnpm typecheck

      - name: Lint
        run: pnpm lint

      - name: Unit tests
        run: pnpm test

      - name: Build
        run: pnpm build

      - name: E2E tests
        run: pnpm test:e2e

  deploy-api:
    needs: test-and-build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

  deploy-web:
    needs: test-and-build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: vercel/vercel-action@main
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
```

### Secrets Management

```bash
# GitHub repo settings → Secrets and variables

NEON_API_KEY=...              # For migrations
FLY_API_TOKEN=...             # Deploy API & Executor
VERCEL_TOKEN=...              # Deploy frontend
DATABASE_URL=...              # For tests
SENTRY_AUTH_TOKEN=...         # Release tracking
```

---

## Monitoring & Observability

### Application Logs (Axiom)

```bash
# Visit https://app.axiom.co
# Create dataset: ai-company-logs
```

**Winston logger setup:**
```typescript
// src/infrastructure/config/logger.ts
import * as axiom from '@axiom-api/axiom';

const axLog = axiom.createClient({ token: env.AXIOM_API_TOKEN });

export const logger = new Logger();
logger.log('Application started', { timestamp: new Date() });
```

### Error Tracking (Sentry)

```bash
# Visit https://sentry.io
# Create project: ai-company-platform
```

**Setup:**
```typescript
// src/main.ts
import * as Sentry from '@sentry/nest';

Sentry.init({
  dsn: env.SENTRY_DSN,
  environment: env.NODE_ENV,
  tracesSampleRate: 1.0,
});

app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.errorHandler());
```

### Metrics (Datadog or Prometheus)

```typescript
// Track key metrics
metrics.gauge('agents.active', activeAgentCount);
metrics.counter('tasks.completed', 1, { agentRole: 'engineer' });
metrics.histogram('execution.duration_ms', executionMs);
```

---

## Scaling Path

### Current (MVP)
- **API:** 1 instance (Railway/Fly.io)
- **DB:** Neon managed (auto-scaling connections)
- **VMs:** 1 per company (auto-hibernates)
- **Traffic:** ~100 RPS capacity

### Phase 2 (Growth)
- **API:** 2-3 instances (load balanced)
- **DB:** Neon + read replicas (for reporting)
- **Cache:** Redis cluster (Upstash)
- **VMs:** Auto-scaling pool per region
- **Traffic:** ~1000 RPS capacity

### Phase 3 (Enterprise)
- **API:** Auto-scaling group (10+ instances)
- **DB:** PostgreSQL cluster (dedicated)
- **Cache:** Redis cluster + distributed caching
- **VMs:** Multi-region deployment
- **CDN:** CloudFront for static assets
- **Traffic:** 10K+ RPS capacity

---

## Rollback Procedures

### API Rollback (Railway)

```bash
# View deployment history
railway logs

# Rollback to previous version
railway rollback [previous-deploy-id]
```

### Executor Rollback (Fly.io)

```bash
# View releases
fly releases list

# Rollback to previous release
fly releases rollback
```

### Database Rollback

```bash
# Restore from Neon snapshot
# In Neon console:
# Branches → main → Restore from backup

# Or manual restore
# psql $DATABASE_URL < backup-2024-03-17.sql
```

---

## Health Checks & Troubleshooting

### API Health

```bash
# Check API status
curl https://api.yourcompany.com/health

# Expected response:
# { "status": "ok", "timestamp": "2026-03-17T12:00:00Z" }
```

### Database Connection

```bash
# SSH into API container
fly ssh console

# Check connection
psql $DATABASE_URL -c "SELECT 1;"
```

### Executor Status

```bash
# Check executor running
curl http://vm-company-1.internal:8080/status

# Expected response:
# { "status": "ready", "adapters": ["claude", "openclaw", "process"] }
```

### Real-time Events

```bash
# Monitor Redis channels
redis-cli SUBSCRIBE "company-*:execution"

# Should see events as tasks execute
```

---

## Checklist: Pre-Launch

- [ ] All 19 phases complete
- [ ] TypeCheck + Lint pass
- [ ] Unit tests >80% coverage
- [ ] E2E tests all green
- [ ] Database migrations tested
- [ ] Secrets in GitHub Actions
- [ ] Monitoring configured (Axiom, Sentry)
- [ ] SSL certificates renewed
- [ ] DNS records pointing to live services
- [ ] Load testing completed
- [ ] Security audit done
- [ ] Backup restoration tested
- [ ] Team trained on ops procedures

---

**Version:** 1.0
**Last Updated:** 2026-03-17
**See Also:** `./system-architecture.md`, `./project-roadmap.md`
