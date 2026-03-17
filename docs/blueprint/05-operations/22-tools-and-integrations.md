# 22 — Tools & Integrations

## Adapter System (9+ Agent Runtimes)

All adapters run inside the Fly.io VM via the Agent Executor. They spawn agent CLI processes, inject API keys as environment variables, and stream results back to the control plane.

### Claude (`claude`)
| Property | Value |
|----------|-------|
| Command | `claude` CLI |
| API Key | `ANTHROPIC_API_KEY` |
| Session | `--context-file` for resume |
| Models | Claude Sonnet, Opus, Haiku |
| Best for | Code, architecture, complex reasoning |

### Codex (`codex`)
| Property | Value |
|----------|-------|
| Command | `codex` CLI |
| API Key | `OPENAI_API_KEY` |
| Session | Implicit cwd-based |
| Models | GPT-4, o-series |
| Best for | Code generation, refactoring |

### Cursor (`cursor`)
| Property | Value |
|----------|-------|
| Command | `cursor-agent` CLI |
| API Key | `CURSOR_API_KEY` |
| Session | `--resume` flag |
| Best for | IDE-integrated coding |

### Gemini (`gemini`)
| Property | Value |
|----------|-------|
| Command | `gemini` CLI |
| API Key | `GOOGLE_API_KEY` |
| Session | `--resume` flag |
| Best for | Research, multimodal tasks |

### OpenCode (`opencode`)
| Property | Value |
|----------|-------|
| Command | `opencode` CLI |
| API Key | Various (provider-dependent) |
| Session | `--session` flag |
| Best for | Open-source model support |

### Pi (`pi`)
| Property | Value |
|----------|-------|
| Command | `pi` CLI |
| API Key | Various |
| Session | `--session` flag |
| Best for | Grok-based tasks |

### OpenClaw Gateway (`openclaw_gateway`)
| Property | Value |
|----------|-------|
| Protocol | WebSocket to remote instance |
| API Key | Via gateway config |
| Session | fixed/issue/run key strategies |
| Best for | Marketing, design, non-code agents |

### Process (`process`)
| Property | Value |
|----------|-------|
| Command | Any shell command |
| Best for | Custom scripts, automation |

### HTTP (`http`)
| Property | Value |
|----------|-------|
| Protocol | POST to webhook URL |
| Best for | External/cloud agent APIs |

## Adding New Adapters

Implement one interface:
```typescript
interface ServerAdapterModule {
  type: string;
  execute(ctx): Promise<AdapterExecutionResult>;
  testEnvironment(ctx): Promise<TestResult>;
  sessionCodec?: AdapterSessionCodec;
  models?: AdapterModel[];
}
```

No core changes needed. Just add to adapter registry. The new adapter runs inside the Fly.io VM just like all others.

## Execution Engine (Fly.io)

All agent execution uses Fly.io Machines as the execution plane. There is a single execution path: control plane dispatches to the Agent Executor on the company's Fly.io VM.

- Sends execution request to Agent Executor on Fly.io VM
- Per-second billing (~$0.008/hour for 2 vCPU)
- 2-5 second VM boot (from hibernated state)
- Persistent volumes for workspace (git repos, files)
- Firecracker isolation (security, per company)
- Auto-hibernate after idle timeout

### Future Execution Backends (Zero Migration)
```
execution/
├── cloud-runner.ts       ← Current (Fly.io)
├── aws-runner.ts         ← Future
├── e2b-runner.ts         ← Future
├── hetzner-runner.ts     ← Future
```

Company config: `{ runnerConfig: { region: "sjc", size: "shared-cpu-2x" } }`
Change provider = change one JSONB field. No schema migration.

## External Services

### Fly.io Machines (Compute)
- REST API for VM lifecycle
- `POST /machines` → boot in ~3 seconds
- Persistent volumes for git repos and workspace
- Per-second billing
- Auto-hibernate support via `machines.stop()`

### PostgreSQL (Database)
- **Neon** or **Supabase** (managed, connection pooling)
- Serverless-friendly, auto-scale
- Automatic backups handled by provider

### Redis (Real-time Events)
- **Upstash** (serverless, per-request billing)
- Used exclusively for pub/sub events (not job queues)
- TLS connection, no persistent storage of sensitive data

### S3 (File Storage)
- **AWS S3** or S3-compatible (MinIO)
- Stores: execution logs, attachments, exported data
- Server-side encryption enabled

### Better Auth (User Authentication)
- Session-based auth with cookies
- OAuth support (Google, GitHub)
- Drizzle adapter for PostgreSQL

### Stripe (Billing — Future)
- Usage-based billing
- Credit packs
- Subscription tiers

## API Key Security

User's LLM API keys flow:
```
User enters key in UI
  → Encrypted (AES-256) → stored in companyApiKeys table
  → Never shown again after entry
  → Decrypted only in server memory during heartbeat
  → Passed to Execution Engine as part of execution request
  → Agent Executor injects as env var to agent process
  → Never stored on VM filesystem
  → Never logged or exposed in events
  → Validated on entry (test API call)
```
