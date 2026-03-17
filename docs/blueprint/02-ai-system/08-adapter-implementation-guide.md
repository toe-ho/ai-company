# 08 — Adapter Implementation Guide

Each adapter handles the specifics of spawning, communicating with, and parsing output from a particular AI agent CLI. This document provides the implementation detail for each adapter.

## Adapter Interface (All Adapters Implement This)

```typescript
interface ServerAdapterModule {
  type: string;

  // Main execution — spawns agent, returns result
  execute(ctx: AdapterExecutionContext): Promise<AdapterExecutionResult>;

  // Test if the agent CLI is installed and configured
  testEnvironment(ctx: AdapterEnvironmentTestContext): Promise<AdapterEnvironmentTestResult>;

  // Serialize/deserialize session state across heartbeats
  sessionCodec?: AdapterSessionCodec;

  // List available models for this adapter
  models?: AdapterModel[];
  listModels?: () => Promise<AdapterModel[]>;

  // Documentation string for adapter-specific config fields
  agentConfigurationDoc?: string;

  // Hook called when an agent hire using this adapter is approved
  onHireApproved?: (payload: any, config: any) => Promise<HireApprovedHookResult>;
}

interface AdapterExecutionContext {
  runId: string;
  agent: { id: string; name: string; role: string; adapterConfig: any };
  runtime: { session: any; taskSessions: any[] };
  workspace: { cwd: string; repoUrl?: string; repoRef?: string };
  context: { taskId?: string; wakeReason?: string; commentId?: string };
  authToken: string;
  apiKey: string;           // User's LLM API key (decrypted)
  apiKeyProvider: string;   // "anthropic", "openai", "google"
  onLog: (event: LogEvent) => void;  // Stream callback
}

interface AdapterExecutionResult {
  exitCode: number | null;
  signal: string | null;
  timedOut: boolean;
  usage: {
    inputTokens: number;
    outputTokens: number;
    cachedInputTokens?: number;
    costUsd: number;
  };
  sessionParams: Record<string, unknown> | null;
  sessionDisplayId: string | null;
  provider: string;
  model: string;
  billingType: "api" | "subscription";
  summary: string;
  resultJson?: any;
}

interface AdapterSessionCodec {
  deserialize(raw: unknown): Record<string, unknown> | null;
  serialize(params: Record<string, unknown> | null): Record<string, unknown> | null;
  getDisplayId?: (params: Record<string, unknown>) => string | null;
}
```

---

## Claude Adapter (`claude`)

### Command Construction
```
Command: claude
Args:
  --model <model>                        (from adapterConfig.model)
  --output-format json                   (always, for structured result parsing)
  --max-turns 100                        (default, configurable)

  IF session.sessionId exists:
    --context-file /tmp/sessions/<sessionId>.json
  IF adapterConfig.effort:
    --effort <low|medium|high>
  IF adapterConfig.dangerouslySkipPermissions:
    --dangerously-skip-permissions
  IF adapterConfig.chrome:
    --chrome

Env var for API key: ANTHROPIC_API_KEY
```

### Output Parsing
Claude outputs JSON blocks to stdout. Look for lines matching:
```json
{ "result": "...", "sessionId": "...", "costUsd": 0.44, "inputTokens": 85000, "outputTokens": 12000 }
```

Parse the **last** JSON block in stdout as the result. Earlier JSON blocks may be intermediate tool-call results.

### Session Codec
```typescript
sessionCodec = {
  deserialize(raw) {
    if (!raw?.sessionId) return null;
    return {
      sessionId: raw.sessionId || raw.session_id,
      cwd: raw.cwd || raw.workdir,
      workspaceId: raw.workspaceId
    };
  },
  serialize(params) {
    if (!params?.sessionId) return null;
    return { sessionId: params.sessionId, cwd: params.cwd, workspaceId: params.workspaceId };
  },
  getDisplayId(params) { return params?.sessionId; }
}
```

### Environment Cleanup
Before spawning, remove these vars from env to prevent "cannot launch inside session" errors:
```
CLAUDE_CODE_SESSION, CLAUDE_CODE_PARENT_SESSION_ID, CLAUDE_CODE_ENTRY_POINT
```

---

## Codex Adapter (`codex`)

### Command Construction
```
Command: codex
Args:
  --model <model>                        (from adapterConfig.model)

  IF adapterConfig.search:
    --search
  IF adapterConfig.dangerouslyBypassApprovalsAndSandbox:
    Append flag

Env var for API key: OPENAI_API_KEY
```

### Output Parsing
Codex outputs JSON objects with `subtype` field:
```json
{ "subtype": "result", "text": "...", "inputTokens": 50000, "outputTokens": 8000 }
```

### Session Codec
Codex uses implicit cwd-based session resume. If the same working directory is used, Codex picks up context from previous runs automatically.
```typescript
sessionCodec = {
  deserialize(raw) {
    return raw?.cwd ? { cwd: raw.cwd } : null;
  },
  serialize(params) {
    return params?.cwd ? { cwd: params.cwd } : null;
  }
}
```

---

## Cursor Adapter (`cursor`)

### Command Construction
```
Command: cursor-agent
Args:
  --model <model>
  --yolo                                 (always auto-added for autonomous execution)

  IF session exists AND session.cwd matches current cwd:
    --resume
  IF adapterConfig.mode:
    --mode <plan|ask>

Env var for API key: CURSOR_API_KEY
```

### Output Parsing
Cursor outputs JSONL (one JSON object per line):
```json
{"type": "text", "content": "Analyzing codebase..."}
{"type": "tool_call", "name": "edit_file", "args": {...}}
{"type": "result", "content": "Done", "tokens": {"input": 60000, "output": 10000}}
```

Parse each line individually. The last `type: "result"` line is the final result.

### Session Codec
```typescript
sessionCodec = {
  deserialize(raw) {
    return raw?.cwd ? { cwd: raw.cwd, resumed: true } : null;
  },
  serialize(params) {
    return params?.cwd ? { cwd: params.cwd } : null;
  }
}
```

---

## Gemini Adapter (`gemini`)

### Command Construction
```
Command: gemini
Args:
  --model <model>

  IF session exists:
    --resume
  IF adapterConfig.sandbox:
    --sandbox

Env var for API key: GOOGLE_API_KEY
```

### Output Parsing
Similar to Claude — look for JSON result blocks in stdout.

### Session Codec
No session ID exposed. Resume via `--resume` flag if cwd matches.

---

## OpenCode Adapter (`opencode`)

### Command Construction
```
Command: opencode
Args:
  --provider <provider>                  (e.g., "anthropic", "openai")
  --model <model>

  IF session exists:
    --session <sessionId>

Env var: varies by provider
```

### Dynamic Model Listing
OpenCode supports `listModels()` — queries available models at runtime.

---

## Pi Adapter (`pi`)

### Command Construction
```
Command: pi
Args:
  --provider <provider>
  --model <model>

  IF session exists:
    --session <sessionId>

Available tools: read, bash, edit, write, grep, find, ls
```

---

## OpenClaw Gateway Adapter (`openclaw_gateway`)

This adapter is fundamentally different — it uses **WebSocket** instead of spawning a process.

### Configuration
```json
{
  "url": "wss://openclaw.example.com/gateway",
  "headers": { "x-openclaw-token": "secret" },
  "clientId": "gateway-client",
  "sessionKeyStrategy": "issue",
  "timeoutSec": 120
}
```

### Execution Protocol
```
1. Connect WebSocket to config.url
2. Authenticate with config.headers
3. Send frame:
   {
     "type": "req",
     "method": "agent.wake",
     "params": {
       "agentId": "...",
       "runId": "...",
       "issueId": "...",
       "wakeReason": "..."
     }
   }
4. Await "agent.ready" event
5. Stream "event" frames (progress, output) → onLog() callback
6. Await "agent.done" frame with result
7. Close WebSocket
```

### Session Key Strategies
- `fixed` — Same session across all runs (agent maintains all state)
- `issue` — One session per issue (resume if same issue)
- `run` — Fresh session every run (no resume)

---

## Process Adapter (`process`)

### Command Construction
```
Command: adapterConfig.command          (e.g., "bash", "python3", "/path/to/script.sh")
Args: adapterConfig.args                (e.g., ["-c", "echo hello"])

Env var: custom from adapterConfig.env
```

Generic adapter for any shell command. No session support. No output parsing beyond exit code.

---

## HTTP Adapter (`http`)

### Execution
```
POST adapterConfig.url
Headers: adapterConfig.headers
Body: {
  runId, agentId, companyId,
  context: { taskId, wakeReason, ... },
  workspace: { cwd, repoUrl, ... }
}

Response 2xx → invocation accepted
Response non-2xx → invocation failed
```

No streaming. No session support. Fire-and-forget webhook.

---

## Adding a New Adapter

1. Create `packages/adapters/<name>/src/`
2. Implement `ServerAdapterModule` interface
3. Add to adapter registry map: `{ "<name>": module }`
4. Add UI config fields in `ui/src/adapters/<name>/config-fields.tsx`
5. Add CLI formatter in `cli/src/adapters/<name>/`
6. Add to shared constants: `AGENT_ADAPTER_TYPES` array
7. Pre-install CLI in Agent Executor Docker image

No core logic changes needed. The adapter registry is the only integration point.

## Per-Adapter Output Parsing Summary

| Adapter | Output Format | Result Detection | Usage Extraction |
|---------|--------------|-----------------|-----------------|
| Claude | JSON blocks in stdout | Last `{ "result": ... }` block | `inputTokens`, `outputTokens`, `costUsd` in result JSON |
| Codex | JSON objects with `subtype` | `subtype: "result"` | `inputTokens`, `outputTokens` in result |
| Cursor | JSONL (one per line) | `type: "result"` line | `tokens` object in result line |
| Gemini | JSON blocks | Last result block | Token counts in result |
| OpenCode | JSON blocks | Result block | Token counts in result |
| Pi | JSON blocks | Result block | Token counts in result |
| OpenClaw | WebSocket frames | `agent.done` frame | Usage in done frame payload |
| Process | Raw text | Exit code only | None |
| HTTP | HTTP response | Status code | None |
