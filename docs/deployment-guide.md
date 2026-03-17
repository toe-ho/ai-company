# Deployment Guide

## Overview

Complete infrastructure setup for deploying the AI Company Platform to production. Covers cloud services, CI/CD pipeline, environment configuration, and monitoring.

## Cloud Infrastructure Architecture

```
┌──────────────────────────────────────────────────────┐
│  Internet                                             │
│  └─ HTTPS → Cloudflare (DNS + SSL)                    │
└───────────────┬──────────────────────────────────────┘
                │
        ┌───────┴────────┐
        │   Control Plane │ (API Server)
        │  (Railway/      │
        │   Fly.io)       │
        └─────┬──────┬────┘
              │      │
    ┌─────────┘      └──────────┐
    │                            │
    ↓                            ↓
┌─────────────┐         ┌──────────────┐
│ PostgreSQL  │         │  Redis       │
│ (Neon)      │         │  (Upstash)   │
│ Multi-      │         │  Pub/sub     │
│ tenant DB   │         │  only        │
└─────────────┘         └──────────────┘

┌──────────────────────────────────────────────────────┐
│  Execution Plane (Fly.io Machines)                    │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │ Company A  │  │ Company B  │  │ Company C  │    │
│  │ VM         │  │ VM         │  │ VM         │    │
│  │ 2-4 CPU    │  │ 2-4 CPU    │  │ 2-4 CPU    │    │
│  │ 4-8 GB RAM │  │ 4-8 GB RAM │  │ 4-8 GB RAM │    │
│  └────────────┘  └────────────┘  └────────────┘    │
└──────────────────────────────────────────────────────┘

┌──────────────┐
│ AWS S3       │
│ Logs,        │
│ Attachments  │
└──────────────┘
```

## Service Providers

| Service | Provider | Cost (Monthly) | Notes |
|---------|----------|----------------|-------|
| **API Server** | Railway or Fly.io | $10-20 | Stateless, auto-scaling |
| **Database** | Neon (PostgreSQL) | $0-25 | Starts free, Pro tier recommended |
| **Redis** | Upstash | $0-10 | Per-request billing, no persistence |
| **Storage** | AWS S3 | $1-5 | Logs, attachments, exports |
| **Domains** | Cloudflare or Namecheap | $0-2 | SSL, DNS, DDoS protection |
| **Monitoring** | Betterstack or Datadog | $0-500 | Optional, free tier available |
| **Per-Company VMs** | Fly.io Machines | $10-20 each | Variable, passed to users |
| **TOTAL (Platform)** | | **$21-62/mo** | Fixed overhead before per-company VMs |

## Prerequisites

- GitHub account (for code + CI/CD)
- Railway or Fly.io account (API hosting)
- Neon account (database)
- Upstash account (Redis)
- AWS account (S3)
- Cloudflare account (DNS, SSL)
- Domain name

## Step 1: Database Setup (Neon)

### Create Neon Project

1. Go to [neon.tech](https://neon.tech)
2. Sign up / log in
3. Create new project
4. Select PostgreSQL 15
5. Choose region (closest to API server)
6. Note connection string: `postgresql://user:password@host/database`

### Run Database Migrations

```bash
# Install Drizzle CLI
npm install -g drizzle-kit

# Set environment variable
export DATABASE_URL="postgresql://user:password@host/database"

# Generate migrations (after schema changes)
drizzle-kit generate:pg

# Run migrations
drizzle-kit migrate:pg

# Verify schema
psql $DATABASE_URL -c "\dt"  # List tables
```

### Connection Pooling

For production, enable PgBouncer (included in Neon Pro):

```bash
# Use pooled connection string
postgresql://user:password@host-pooler/database?sslmode=require
```

### Backup Strategy

Neon provides automated daily backups. For critical data:

```bash
# Manual backup
pg_dump $DATABASE_URL > backup-$(date +%Y%m%d).sql

# Restore from backup
psql $DATABASE_URL < backup-YYYYMMDD.sql
```

## Step 2: Redis Setup (Upstash)

### Create Upstash Database

1. Go to [upstash.com](https://upstash.com)
2. Sign up / log in
3. Create new Redis database
4. Select region (same as API server region)
5. Copy connection string: `redis://default:password@host:port`

### Configure Redis Client

```bash
# Backend environment variable
REDIS_URL="redis://default:password@host:port"

# Test connection
redis-cli -u $REDIS_URL PING
# Should return: PONG
```

**Note:** Upstash is pub/sub only (no persistence needed). Perfect for real-time events.

## Step 3: S3 Bucket Setup (AWS)

### Create S3 Bucket

1. Go to AWS Console
2. Navigate to S3
3. Create bucket: `ai-platform-{environment}-{region}`
4. Block all public access (keep private)
5. Enable versioning (for data recovery)
6. Enable server-side encryption

### IAM Credentials

Create IAM user with S3 permissions:

```bash
# Permissions policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::ai-platform-prod/*",
        "arn:aws:s3:::ai-platform-prod"
      ]
    }
  ]
}
```

### Environment Variables

```bash
AWS_ACCESS_KEY_ID="..."
AWS_SECRET_ACCESS_KEY="..."
AWS_S3_BUCKET="ai-platform-prod"
AWS_S3_REGION="us-east-1"
```

## Step 4: API Server Deployment (Railway)

### Option A: Railway (Recommended for Simplicity)

#### Setup

1. Go to [railway.app](https://railway.app)
2. Sign up / log in
3. Create new project
4. Connect GitHub repository
5. Select repo and branch

#### Configuration

1. Add environment variables:
   ```
   DATABASE_URL=postgresql://...
   REDIS_URL=redis://...
   AWS_ACCESS_KEY_ID=...
   AWS_SECRET_ACCESS_KEY=...
   AWS_S3_BUCKET=...
   NODE_ENV=production
   FLY_API_TOKEN=... (for Fly.io integration)
   JWT_SECRET=... (generate: openssl rand -base64 32)
   AUTH_SECRET=... (generate: openssl rand -base64 32)
   ```

2. Build command:
   ```bash
   pnpm install && pnpm build
   ```

3. Start command:
   ```bash
   node dist/apps/backend/main.js
   ```

4. Port: `3100`

#### Deploy

- Railway auto-deploys on GitHub push
- Monitor logs in Railway dashboard
- Set up auto-scaling (Railway Pro)

#### Cost
- Hobby: $5/month
- Pro: $20/month (recommended)

### Option B: Fly.io (Alternative, More Control)

#### Setup

```bash
# Install Fly CLI
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Create app in project directory
fly launch --org your-org --name ai-platform-api
```

#### Configuration

Create `fly.toml`:

```toml
app = "ai-platform-api"
primary_region = "sjc"

[build]
builder = "paas"
buildpacks = ["heroku/nodejs"]

[[services]]
protocol = "tcp"
internal_port = 3100
processes = ["app"]

[services.ports]
ports = [80, 443]

[env]
DATABASE_URL = ""
REDIS_URL = ""
NODE_ENV = "production"
```

#### Deploy

```bash
# Set secrets
fly secrets set DATABASE_URL="..." REDIS_URL="..." ...

# Deploy
fly deploy

# View logs
fly logs

# Scale replicas
fly scale count 3
```

#### Cost
- Shared CPU: ~$5-10/month per instance
- Dedicated CPU: ~$15-50/month per instance

## Step 5: Fly.io Setup (Per-Company VMs)

### Fly.io Account & API Token

1. Create Fly.io account at [fly.io](https://fly.io)
2. Create organization
3. Generate API token: Settings → Tokens → Create Org Token
4. Save token to environment: `FLY_API_TOKEN=...`

### Fly.io Machine Configuration

```javascript
// In backend environment variables:
FLY_ORG_SLUG = "your-org";
FLY_APP_NAME_PREFIX = "ai-company"; // Apps named: ai-company-{companyId}
FLY_REGION = "sjc"; // Closest to your users
FLY_CPU_CORES = 2; // 2-4 CPU cores
FLY_MEMORY_MB = 4096; // 4-8 GB RAM
FLY_VOLUME_GB = 10; // Persistent volume size
```

### Test VM Creation

```bash
# Test Fly.io API
curl -H "Authorization: Bearer $FLY_API_TOKEN" \
  https://api.machines.dev/v1/apps

# Should list apps in your organization
```

## Step 6: GitHub Actions CI/CD

### Create Workflow File

`.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - uses: pnpm/action-setup@v2

      - uses: actions/setup-node@v3
        with:
          node-version: 20
          cache: pnpm

      - run: pnpm install

      - run: pnpm typecheck

      - run: pnpm lint

      - run: pnpm test

      - run: pnpm build

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - uses: pnpm/action-setup@v2

      - uses: actions/setup-node@v3
        with:
          node-version: 20
          cache: pnpm

      - run: pnpm install && pnpm build

      - name: Deploy to Railway
        run: |
          npm install -g @railway/cli
          railway up --service backend
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
```

### Secrets

Add to GitHub repository settings:

```
RAILWAY_TOKEN = "..." (from Railway dashboard)
FLY_API_TOKEN = "..." (from Fly.io)
```

## Step 7: Environment Configuration

### Development (.env.local)

```bash
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/ai_platform

# Redis
REDIS_URL=redis://localhost:6379

# AWS S3
AWS_S3_BUCKET=ai-platform-dev
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...

# Fly.io
FLY_API_TOKEN=...

# Auth
JWT_SECRET=$(openssl rand -base64 32)
AUTH_SECRET=$(openssl rand -base64 32)

# API
API_URL=http://localhost:3100
WEB_URL=http://localhost:5173
```

### Production (.env.production)

```bash
NODE_ENV=production

# Database (from Neon)
DATABASE_URL=postgresql://...

# Redis (from Upstash)
REDIS_URL=redis://...

# AWS S3
AWS_S3_BUCKET=ai-platform-prod
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...

# Fly.io
FLY_API_TOKEN=...
FLY_ORG_SLUG=your-org

# Auth
JWT_SECRET=... (from secure key manager)
AUTH_SECRET=... (from secure key manager)

# API
API_URL=https://api.youromain.com
WEB_URL=https://youromain.com

# Logging
LOG_LEVEL=info
```

### Secure Secrets Management

**DO NOT** commit `.env` files. Use:

- **Railway:** Built-in environment variables in dashboard
- **Fly.io:** `fly secrets set KEY=VALUE`
- **GitHub Actions:** Repository secrets
- **AWS Secrets Manager:** For sensitive keys

```bash
# Railway example
railway variables set DATABASE_URL="..."

# Fly.io example
fly secrets set --app ai-platform-api DATABASE_URL="..."
```

## Step 8: Domain & SSL

### Cloudflare Setup

1. Add domain to Cloudflare (transfer DNS)
2. Update nameservers at domain registrar
3. Create DNS records:

```
Type    Name    Value                TTL
A       @       Railway IP address   Auto
CNAME   api     your-api.railway.app Auto
CNAME   www     your-domain.com      Auto
```

4. Enable SSL (Cloudflare → SSL/TLS → Full (Strict))

### Redirect to API

```
api.yourdomain.com → Railway/Fly.io API server
yourdomain.com → Vercel/Netlify (web frontend)
```

## Step 9: Monitoring & Alerts

### Application Monitoring (Betterstack)

```bash
# 1. Sign up at betterstack.com
# 2. Create monitors for API endpoints

# Health check endpoint
GET https://api.yourdomain.com/api/internal/health

# Response:
{
  "status": "ok",
  "timestamp": "2024-03-17T10:30:00Z",
  "database": "connected",
  "redis": "connected"
}
```

### Database Monitoring (Neon)

- **CPU & Memory:** Neon dashboard
- **Query Performance:** Neon's built-in analytics
- **Replication Lag:** Query `pg_stat_replication`

### Error Tracking (Optional: Sentry)

```typescript
// In NestJS setup
import * as Sentry from "@sentry/node";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
});
```

### Logs (Structured with Pino)

```typescript
// All logs are structured JSON
// Automatically captured by cloud platform

// Example:
{
  "level": "info",
  "timestamp": "2024-03-17T10:30:00Z",
  "service": "backend",
  "request_id": "abc123",
  "user_id": "user1",
  "action": "create_company",
  "status": "success"
}
```

## Step 10: Pre-Launch Checklist

### Infrastructure
- [ ] Database (Neon) created and tested
- [ ] Redis (Upstash) created and tested
- [ ] S3 bucket created with correct permissions
- [ ] Fly.io account with API token
- [ ] Domain registered and DNS configured
- [ ] SSL certificate active (Cloudflare)

### Deployment
- [ ] GitHub Actions CI/CD pipeline working
- [ ] API server deployed and accessible
- [ ] Web frontend deployed
- [ ] Database migrations run successfully
- [ ] Environment variables set correctly
- [ ] Health endpoint responding

### Configuration
- [ ] API_URL and WEB_URL correct in all environments
- [ ] Secrets stored securely (not in git)
- [ ] Logging configured (Pino)
- [ ] Error tracking setup (Sentry optional)
- [ ] Monitoring alerts configured

### Testing
- [ ] Unit tests passing
- [ ] Integration tests passing
- [ ] E2E tests passing (Playwright)
- [ ] Manual testing of critical flows
- [ ] Load testing (k6 or similar)
- [ ] Security audit (penetration testing optional)

### Documentation
- [ ] API documentation (Swagger/OpenAPI)
- [ ] User onboarding guide
- [ ] Admin documentation
- [ ] Runbook for common issues
- [ ] Escalation procedures

### Production Readiness
- [ ] Database backups automated
- [ ] Disaster recovery plan written
- [ ] Monitoring & alerts active
- [ ] Support procedures in place
- [ ] Rollback plan documented

## Deployment Process

### Step-by-Step Deploy

```bash
# 1. Verify all tests pass locally
pnpm test
pnpm test:e2e

# 2. Commit and push to main (triggers CI/CD)
git add .
git commit -m "feat: new feature"
git push origin main

# 3. GitHub Actions runs automatically
# - Installs dependencies
# - Runs linting & typecheck
# - Runs tests
# - Builds (if all pass)
# - Deploys to Railway/Fly.io

# 4. Monitor deployment
# Railway: dashboard.railway.app
# Fly.io: fly logs

# 5. Verify in production
curl https://api.yourdomain.com/api/internal/health
```

### Rollback Procedure

If deployment breaks production:

```bash
# Railway: Redeploy previous version from dashboard
# Fly.io: Roll back to previous release
fly releases list --app ai-platform-api
fly releases rollback --app ai-platform-api

# Or manually revert commit
git revert HEAD
git push origin main
# CI/CD will deploy the reverted version
```

## Database Migrations

### Running Migrations

```bash
# On every deployment, migrations run automatically
# Or manually:

pnpm migrate

# Or with Drizzle CLI:
drizzle-kit migrate:pg --config drizzle.config.ts
```

### Creating New Migration

```bash
# After updating schema in src/infrastructure/database/schemas/
# Generate migration:
drizzle-kit generate:pg

# Review generated SQL in migrations/
# Then run:
pnpm migrate
```

## Troubleshooting

### Database Connection Fails

```bash
# Test connection
psql $DATABASE_URL -c "SELECT 1"

# Check firewall rules (Neon)
# Verify IP whitelist in Neon dashboard

# Test with test client
node -e "
const { db } = require('./dist/infrastructure/db');
db.query.users.findFirst().then(() => console.log('OK'))
"
```

### Redis Connection Fails

```bash
# Test connection
redis-cli -u $REDIS_URL PING

# Check Upstash dashboard
# Verify token is correct
```

### Fly.io VM Creation Fails

```bash
# Check API token
fly auth whoami

# Check org
fly orgs list

# Test API directly
curl -H "Authorization: Bearer $FLY_API_TOKEN" \
  https://api.machines.dev/v1/apps
```

### High Database Latency

```bash
# Use pooled connection string (Neon Pro)
# Enable read replicas (if using Neon Pro)
# Add indexes on frequently queried columns
# Profile slow queries in Neon dashboard
```

### Out of Memory (OOM)

```bash
# Increase Railway/Fly.io instance size
# Enable autoscaling
# Profile memory usage (pino + Clinic.js)
# Reduce Redis pub/sub message size
```

## Cost Optimization

### Minimize Infrastructure Costs

1. **Auto-scaling:** Only run replicas when needed
2. **Database:** Use Neon free tier initially, upgrade later
3. **Redis:** Upstash per-request billing is cheaper than fixed
4. **S3:** Use lifecycle policies to auto-delete old logs
5. **Fly.io:** Use shared CPU for VMs when possible

### Monitor Costs

- Railway: Dashboard shows usage
- Neon: Per-query metrics in dashboard
- AWS: Set up cost alerts (CloudWatch)
- Fly.io: Track VM creation frequency

### Billing Strategy

- **To users:** $49-99/month platform fee + compute markup
- **Internal:** Fixed overhead ~$200/mo + $10-20 per company VM

---

## Production Launch Checklist

- [ ] All infrastructure provisioned
- [ ] CI/CD pipeline tested
- [ ] Database backups verified
- [ ] Monitoring & alerts active
- [ ] SSL certificate valid
- [ ] API endpoints responding
- [ ] Web frontend accessible
- [ ] Test heartbeat execution works
- [ ] Cost tracking accurate
- [ ] Budget limits enforced
- [ ] Real-time events flowing
- [ ] Documentation complete
- [ ] Support channels ready
- [ ] Status page created
- [ ] Incident response plan ready

**Launch: Ready when all items checked.**
