# 02 — AI Architecture

## Important Distinction

This platform is NOT an AI framework, LLM wrapper, or prompt engine. It does not call LLM APIs directly, build models, or manage prompt chains.

It is a **control plane** that orchestrates external AI agents. The "AI architecture" is the **Execution Engine + Adapter system** — how we spawn, monitor, communicate with, and manage diverse AI agent runtimes on cloud VMs.

## Architecture Diagram

```
User Action (Dashboard)
       │
       ▼
┌──────────────────────────────────────┐
│  CONTROL PLANE (API Server)           │
│                                       │
│  ┌──────────────────────────────┐    │
│  │  Heartbeat Service            │    │  ← Orchestrator
│  │                               │    │
│  │  1. Validate agent state      │    │
│  │  2. Retrieve user's API key   │    │  ← From encrypted vault
│  │  3. Build execution context   │    │
│  │  4. Create agent JWT          │    │
│  │  5. Ensure VM running         │    │  ← Via Provisioner
│  │  6. Dispatch to Execution     │    │
│  │     Engine                    │    │
│  │  7. Stream results            │    │
│  │  8. Persist run data          │    │
│  └──────────┬───────────────────┘    │
│             │                         │
│  ┌──────────▼───────────────────┐    │
│  │  Execution Engine             │    │
│  │  (CloudRunner → Fly.io VM)   │    │
│  │                               │    │
│  │  HTTP POST execution request  │    │
│  │  SSE stream results back      │    │
│  └──────────┬───────────────────┘    │
│             │                         │
│  ┌──────────▼───────────────────┐    │
│  │  Adapter Registry             │    │
│  │                               │    │
│  │  claude     → claude CLI      │    │
│  │  codex      → codex CLI       │    │
│  │  cursor     → cursor CLI      │    │
│  │  gemini     → gemini CLI      │    │
│  │  opencode   → opencode CLI    │    │
│  │  pi         → pi CLI          │    │
│  │  openclaw_gw→ WebSocket       │    │
│  │  process    → shell cmd       │    │
│  │  http       → webhook         │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  EXECUTION PLANE (Fly.io VM)          │
│                                       │
│  Agent Executor receives:             │
│  - Environment vars (task context)    │
│  - Stdin prompt                       │
│  - Skill files (task protocol)        │
│  - User's API key (from vault)        │
│  - JWT for callbacks                  │
│                                       │
│  Agent does:                          │
│  - Checkout task (atomic)             │
│  - Read context                       │
│  - Do domain work (code, design, etc) │
│  - Update status                      │
│  - Post comments                      │
│  - Delegate sub-tasks                 │
│  - Exit                               │
└──────────────────────────────────────┘
```

## Two-Layer Abstraction: Execution Engine + Adapter

```
Execution Engine = WHERE and HOW the agent runs
  └── CloudRunner → Fly.io VM (HTTP to Agent Executor)

Adapter = WHICH agent and HOW to talk to it
  ├── claude      → spawn `claude` CLI
  ├── codex       → spawn `codex` CLI
  ├── cursor      → spawn `cursor-agent` CLI
  └── ... (9 total)
```

**They're independent.** The Claude adapter runs inside any Fly.io VM — same adapter code regardless of which VM handles the execution.

### Execution Engine Interface
```typescript
interface ExecutionEngine {
  execute(request: ExecutionRequest): AsyncIterable<ExecutionEvent>;
  cancel(runId: string): Promise<void>;
}
```

### Adapter Interface
```typescript
interface ServerAdapterModule {
  type: string;
  execute(ctx: AdapterExecutionContext): Promise<AdapterExecutionResult>;
  testEnvironment(ctx): Promise<AdapterEnvironmentTestResult>;
  sessionCodec?: AdapterSessionCodec;  // Persist state across heartbeats
  models?: AdapterModel[];
  listModels?: () => Promise<AdapterModel[]>;
}
```

## How Cloud Execution Works (Fly.io)

```
Control Plane                          Fly.io VM
    │                                     │
    ├─ Provisioner ensures VM running     │
    │                                     │
    ├─ Execution Engine sends HTTP ──────►│
    │  { runId, agent, adapter, context,  │
    │    apiKey (from vault), env }        │
    │                                     │
    │                               ┌─────┤
    │                               │ Agent Executor
    │                               │ (lightweight Node.js process)
    │                               │     │
    │                               │     ├─ Resolve adapter
    │                               │     ├─ Inject API key as env var
    │                               │     ├─ Spawn agent process
    │                               │     ├─ Capture stdout/stderr
    │                               │     │
    │  ◄── SSE stream (events) ─────┤     │
    │                               │     │
    │  ◄── Final result ────────────┤     │
    │                               └─────┤
    │                                     │
    ├─ Persist results to DB              │
    ├─ Update cost tracking               │
    ├─ Publish live events                │
    └─ Hibernate VM if idle               │
```

## API Key Flow (Critical for Security)

```
1. User enters API key in webapp
2. Key encrypted with AES-256, stored in companyApiKeys table
3. When heartbeat fires:
   a. Control plane retrieves encrypted key
   b. Decrypts key in memory
   c. Passes to Execution Engine as part of execution request
   d. Agent Executor injects as ANTHROPIC_API_KEY env var
   e. Agent process uses key to call LLM API
4. Key NEVER stored on VM filesystem
5. Key NEVER logged or exposed in events
```

## Session Persistence (State Across Heartbeats)

Each adapter has a `sessionCodec` to serialize/deserialize state:
```typescript
interface AdapterSessionCodec {
  deserialize(raw: unknown): Record<string, unknown> | null;
  serialize(params: Record<string, unknown> | null): Record<string, unknown> | null;
}
```

Stored in `agentTaskSessions` table. Restored on next heartbeat. Enables agents to resume conversations without starting over.

## Safety Controls

| Control | How |
|---------|-----|
| Budget hard cap | Agent auto-paused at 100% monthly budget |
| Timeout | SIGTERM → grace → SIGKILL per heartbeat |
| Orphan reaping | Stale runs (>5 min) marked failed |
| Single concurrent | Max 1 heartbeat per agent (V1) |
| Atomic checkout | 409 on double-work |
| API key isolation | Keys never on disk, encrypted at rest |
| VM isolation | Firecracker-based (Fly.io) per company |
| Kill switch | One-click pause entire company |

## 9+ Adapter Types

| Adapter | Method | Session Resume | API Key Env Var |
|---------|--------|---------------|----------------|
| `claude` | Spawn `claude` CLI | `--context-file` | `ANTHROPIC_API_KEY` |
| `codex` | Spawn `codex` CLI | cwd match | `OPENAI_API_KEY` |
| `cursor` | Spawn `cursor-agent` | `--resume` | `CURSOR_API_KEY` |
| `gemini` | Spawn `gemini` CLI | `--resume` | `GOOGLE_API_KEY` |
| `opencode` | Spawn `opencode` | `--session` | various |
| `pi` | Spawn `pi` CLI | `--session` | various |
| `openclaw_gateway` | WebSocket | fixed/issue/run key | via gateway config |
| `process` | Shell command | None | custom |
| `http` | POST webhook | None | custom |
