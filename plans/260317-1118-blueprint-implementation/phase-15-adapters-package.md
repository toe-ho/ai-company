# Phase 15: Adapters Package

## Context Links
- [08 - Adapter Implementation Guide](../../docs/blueprint/02-ai-system/08-adapter-implementation-guide.md)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 2h
- **Description:** Implement 3 adapter types for V1 (Claude + Process + OpenClaw Gateway) in `packages/adapters` and shared utilities in `packages/adapter-utils`. Remaining 6 adapters deferred to V2.
<!-- Updated: Validation Session 1 - Reduced from 9 to 3 adapters (Claude + Process + OpenClaw) -->

## Key Insights
- Each adapter implements `ServerAdapterModule` interface
- Adapters handle: command construction, output parsing, session codec, environment cleanup
- Claude adapter is highest priority (primary agent CLI)
- OpenClaw Gateway is unique: WebSocket instead of process spawn
- Process and HTTP adapters are generic fallbacks
- adapter-utils provides shared output parsing, base helpers

## Architecture
```
packages/adapter-utils/src/
├── index.ts
├── output-parser.ts              # Parse JSON from stdout
├── process-helper.ts             # Spawn helper, env cleanup
├── session-file-manager.ts       # Read/write session files
└── api-key-env-mapper.ts         # Map provider -> env var name

packages/adapters/src/
├── index.ts                      # Barrel + registry
├── registry.ts                   # AdapterRegistry map
├── types.ts                      # ServerAdapterModule interface (re-export from shared)
├── claude/
│   ├── claude-adapter.ts
│   └── claude-session-codec.ts
├── codex/
│   ├── codex-adapter.ts
│   └── codex-session-codec.ts
├── cursor/
│   ├── cursor-adapter.ts
│   └── cursor-session-codec.ts
├── gemini/
│   ├── gemini-adapter.ts
│   └── gemini-session-codec.ts
├── opencode/
│   ├── opencode-adapter.ts
│   └── opencode-session-codec.ts
├── pi/
│   ├── pi-adapter.ts
│   └── pi-session-codec.ts
├── openclaw-gateway/
│   ├── openclaw-gateway-adapter.ts
│   └── openclaw-session-codec.ts
├── process/
│   └── process-adapter.ts
└── http/
    └── http-adapter.ts
```

## Related Code Files

### Files to Create
~25 files across both packages.

## Implementation Steps

1. **Create adapter-utils**

   **output-parser.ts**:
   - `parseLastJsonBlock(stdout: string): unknown | null` -- find last `{ ... }` JSON block in output
   - `parseJsonLines(stdout: string): unknown[]` -- parse JSONL (one JSON per line)
   - `extractUsage(result: unknown): { inputTokens, outputTokens, costUsd }`

   **process-helper.ts**:
   - `cleanEnv(env: Record<string, string>, keysToRemove: string[]): Record<string, string>`
   - `buildApiKeyEnv(provider: string, key: string): Record<string, string>`

   **session-file-manager.ts**:
   - `writeSessionFile(sessionId: string, data: unknown): string` -- writes to /tmp/sessions/, returns path
   - `readSessionFile(path: string): unknown | null`
   - `cleanSessionFiles()` -- remove old session files

   **api-key-env-mapper.ts**:
   - `{ anthropic: 'ANTHROPIC_API_KEY', openai: 'OPENAI_API_KEY', google: 'GOOGLE_API_KEY' }`

2. **Create Claude adapter** (highest priority)
   ```typescript
   export const claudeAdapter: ServerAdapterModule = {
     type: 'claude',
     async execute(ctx) {
       const args = ['--model', ctx.agent.adapterConfig.model, '--output-format', 'json', '--max-turns', '100'];
       if (ctx.runtime.session?.sessionId) {
         const path = writeSessionFile(ctx.runtime.session.sessionId, ctx.runtime.session);
         args.push('--context-file', path);
       }
       if (ctx.agent.adapterConfig.effort) args.push('--effort', ctx.agent.adapterConfig.effort);
       if (ctx.agent.adapterConfig.dangerouslySkipPermissions) args.push('--dangerously-skip-permissions');

       const env = cleanEnv(process.env, ['CLAUDE_CODE_SESSION', 'CLAUDE_CODE_PARENT_SESSION_ID', 'CLAUDE_CODE_ENTRY_POINT']);
       env.ANTHROPIC_API_KEY = ctx.apiKey;

       // Spawn, stream, collect result
       // Parse last JSON block for result + usage
     },
     sessionCodec: claudeSessionCodec,
     models: [{ id: 'claude-sonnet-4-20250514', name: 'Claude Sonnet 4' }, ...],
   };
   ```

   **claude-session-codec.ts**: Per blueprint 08 -- serialize/deserialize sessionId + cwd + workspaceId

3. **Create Codex adapter**
   - Command: `codex --model <model>`
   - Optional: `--search`, `--dangerouslyBypassApprovalsAndSandbox`
   - Env: OPENAI_API_KEY
   - Parse JSON with `subtype: "result"` for final result
   - Session: cwd-based implicit resume

4. **Create Cursor adapter**
   - Command: `cursor-agent --model <model> --yolo`
   - Optional: `--resume`, `--mode <plan|ask>`
   - Env: CURSOR_API_KEY
   - Parse JSONL, last `type: "result"` line
   - Session: cwd-based resume

5. **Create Gemini adapter**
   - Command: `gemini --model <model>`
   - Optional: `--resume`, `--sandbox`
   - Env: GOOGLE_API_KEY
   - Parse JSON blocks similar to Claude

6. **Create OpenCode adapter**
   - Command: `opencode --provider <provider> --model <model>`
   - Optional: `--session <sessionId>`
   - Dynamic model listing support

7. **Create Pi adapter**
   - Command: `pi --provider <provider> --model <model>`
   - Optional: `--session <sessionId>`

8. **Create OpenClaw Gateway adapter** (WebSocket-based)
   ```typescript
   export const openclawGatewayAdapter: ServerAdapterModule = {
     type: 'openclaw_gateway',
     async execute(ctx) {
       const ws = new WebSocket(ctx.agent.adapterConfig.url, {
         headers: ctx.agent.adapterConfig.headers,
       });
       // Send agent.wake frame
       // Listen for events, forward to ctx.onLog
       // Wait for agent.done frame
       // Close WebSocket
       // Return result
     },
     sessionCodec: openclawSessionCodec,
   };
   ```
   - Session key strategies: fixed, issue, run

9. **Create Process adapter** (generic)
   - Command + args from adapterConfig
   - No session, no output parsing beyond exit code
   - Env: custom from adapterConfig.env

10. **Create HTTP adapter** (webhook)
    - POST to adapterConfig.url with request payload
    - No streaming, no session
    - Returns success/failure based on HTTP status

11. **Create `registry.ts`**
    ```typescript
    export const adapterRegistry: Record<string, ServerAdapterModule> = {
      claude: claudeAdapter,
      codex: codexAdapter,
      cursor: cursorAdapter,
      gemini: geminiAdapter,
      opencode: opencodeAdapter,
      pi: piAdapter,
      openclaw_gateway: openclawGatewayAdapter,
      process: processAdapter,
      http: httpAdapter,
    };
    ```

## Todo List
- [ ] adapter-utils: output parser, process helper, session file manager, env mapper
- [ ] Claude adapter + session codec
- [ ] Process adapter (generic fallback)
- [ ] OpenClaw Gateway adapter (WebSocket)
- [ ] ~~Codex adapter~~ (deferred to V2)
- [ ] ~~Cursor adapter~~ (deferred to V2)
- [ ] ~~Gemini adapter~~ (deferred to V2)
- [ ] ~~OpenCode adapter~~ (deferred to V2)
- [ ] ~~Pi adapter~~ (deferred to V2)
- [ ] ~~HTTP adapter~~ (deferred to V2)
- [ ] Adapter registry
- [ ] Barrel exports
- [ ] typecheck passes

## Success Criteria
- All 9 adapters implement ServerAdapterModule interface
- Claude adapter correctly builds command with all flag variants
- Session codecs roundtrip (serialize -> deserialize = original)
- Adapter registry resolves all types
- `turbo typecheck` passes

## Risk Assessment
- **CLI version changes:** Adapter args may need updating as CLIs evolve; document expected versions
- **WebSocket adapter complexity:** OpenClaw Gateway is fundamentally different; may need additional error handling
- **Output parsing fragility:** JSON blocks in stdout may be mixed with other output; parser must be robust

## Security Considerations
- API keys passed only as env vars in spawned process
- Scrub API keys from stdout/stderr output before streaming
- WebSocket connections use auth headers from config (not user input)

## Next Steps
- Phase 14 executor imports adapter registry for command building
