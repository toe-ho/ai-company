# 23 — Config & Environment

## Environment Variables

### Server
| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3100` | Server listen port |
| `HOST` | `0.0.0.0` | Bind host |
| `NODE_ENV` | `production` | Environment |

### Database
| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string (Neon/Supabase) |

### Auth
| Variable | Required | Description |
|----------|----------|-------------|
| `AUTH_SECRET` | Yes | Session signing secret |
| `AUTH_URL` | Yes | Auth callback URL |
| `AGENT_JWT_SECRET` | Yes | Agent JWT signing secret |
| `AGENT_JWT_TTL_SECONDS` | `172800` (48h) | Agent JWT time-to-live |

### Fly.io (Execution Plane)
| Variable | Required | Description |
|----------|----------|-------------|
| `FLY_API_TOKEN` | Yes | Fly.io API authentication |
| `FLY_APP_NAME` | Yes | Fly.io app for company VMs |
| `FLY_REGION` | `sjc` | Default VM region |
| `FLY_VM_SIZE` | `shared-cpu-2x` | Default VM size |
| `FLY_IDLE_TIMEOUT_MIN` | `10` | Hibernate after N minutes idle |

### Redis
| Variable | Required | Description |
|----------|----------|-------------|
| `REDIS_URL` | Yes | Redis connection (Upstash) |

### Storage
| Variable | Required | Description |
|----------|----------|-------------|
| `S3_BUCKET` | Yes | S3 bucket name |
| `S3_REGION` | Yes | S3 region |
| `S3_ENDPOINT` | No | Custom endpoint (MinIO) |

### Encryption
| Variable | Required | Description |
|----------|----------|-------------|
| `ENCRYPTION_KEY` | Yes | 32-byte AES key for API key vault |

### Features
| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_AGENTS_PER_COMPANY` | `20` | Self-hiring limit |
| `MAX_HIRE_DEPTH` | `3` | Max org tree depth from self-hiring |

## Required Environment Setup

All deployments require these variables. Generate secrets with `openssl rand -base64 32`.

```bash
# Database
DATABASE_URL=postgres://...          # Neon/Supabase connection string

# Auth
AUTH_SECRET=<generate>               # openssl rand -base64 32
AUTH_URL=https://your-domain.com
AGENT_JWT_SECRET=<generate>          # openssl rand -base64 32

# Fly.io (execution plane)
FLY_API_TOKEN=<from-fly.io>         # fly auth token
FLY_APP_NAME=my-app-vms

# Redis
REDIS_URL=redis://...                # Upstash connection URL

# Storage
S3_BUCKET=my-bucket
S3_REGION=us-east-1

# Encryption
ENCRYPTION_KEY=<generate>            # openssl rand -base64 32
```

## Executor Environment Variables (Injected at Runtime)

These variables are injected into the Fly.io VM by the provisioner at boot time. Not committed to `.env` files.

| Variable | Description |
|----------|-------------|
| `CONTROL_PLANE_URL` | Backend API base URL for agent callbacks |
| `COMPANY_ID` | UUID of the company this VM serves |
| `AGENT_JWT_SECRET` | Secret for verifying agent JWTs |
| `ANTHROPIC_API_KEY` | User's LLM key (if adapter is Claude) |
| `OPENAI_API_KEY` | User's LLM key (if adapter is Codex/OpenAI) |
| `GOOGLE_API_KEY` | User's LLM key (if adapter is Gemini) |

## Docker

```dockerfile
FROM node:lts-slim AS build
RUN pnpm install && pnpm build

FROM node:lts-slim AS production
# Pre-install agent CLIs: claude, codex, opencode
COPY --from=build /app/dist ./
EXPOSE 3100
CMD ["node", "server/dist/index.js"]
```

## Per-Company Configuration

Stored in database (not env vars):

```json
{
  "name": "My AI Startup",
  "runnerConfig": {
    "region": "sjc",
    "size": "shared-cpu-2x",
    "volumeSize": "10gb",
    "idleTimeoutMin": 10
  },
  "budgetMonthlyCents": 200000,
  "requireBoardApprovalForNewAgents": true,
  "maxAgents": 20
}
```

## Per-Agent Configuration

```json
{
  "adapterType": "claude",
  "adapterConfig": {
    "model": "claude-sonnet-4-20250514",
    "effort": "high",
    "timeoutSec": 600,
    "env": {
      "NODE_ENV": "production"
    }
  },
  "runtimeConfig": {
    "heartbeatEnabled": true,
    "heartbeatIntervalSec": 300,
    "maxConcurrentRuns": 1
  },
  "budgetMonthlyCents": 5000
}
```
