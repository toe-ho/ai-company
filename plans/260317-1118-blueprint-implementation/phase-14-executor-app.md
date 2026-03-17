# Phase 14: Executor App

## Context Links
- [07 - Agent Executor Spec](../../docs/blueprint/02-ai-system/07-agent-executor-spec.md)
- [08 - Adapter Implementation Guide](../../docs/blueprint/02-ai-system/08-adapter-implementation-guide.md)
- [14 - Monorepo Setup](../../docs/blueprint/03-architecture/14-monorepo-setup-guide.md) (Docker section)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 3h
- **Description:** Express server running inside Fly.io VMs. Receives execution requests, spawns agent processes, streams results back via SSE.

## Key Insights
- Lightweight Express server on port 8080 (not NestJS -- too heavy for VM)
- One execution at a time per VM (409 on concurrent request)
- SSE streaming: `Content-Type: text/event-stream`
- 10-step execution pipeline: validate -> workspace -> env -> skills -> command -> stdin -> spawn -> stream -> timeout -> result
- Never store API keys to disk; env vars only in child process
- Clean up child processes on shutdown (SIGTERM handler)

## Architecture
```
apps/executor/
├── src/
│   ├── main.ts                          # Express server bootstrap
│   ├── routes/
│   │   ├── execute-route.ts             # POST /execute -> SSE stream
│   │   ├── cancel-route.ts             # POST /cancel -> SIGTERM child
│   │   └── health-route.ts             # GET /health
│   ├── pipeline/
│   │   ├── execution-pipeline.ts       # Orchestrates 10-step pipeline
│   │   ├── request-validator.ts        # Step 1: validate request
│   │   ├── workspace-resolver.ts       # Step 2: git clone/checkout
│   │   ├── environment-builder.ts      # Step 3: merge env vars
│   │   ├── skill-linker.ts            # Step 4: symlink skills
│   │   ├── command-builder.ts         # Step 5: adapter-specific command
│   │   ├── prompt-renderer.ts         # Step 6: render stdin prompt
│   │   ├── process-spawner.ts         # Step 7: spawn child process
│   │   ├── output-streamer.ts         # Step 8: pipe stdout/stderr -> SSE
│   │   ├── timeout-enforcer.ts        # Step 9: SIGTERM/SIGKILL
│   │   └── result-collector.ts        # Step 10: collect exit code + usage
│   ├── sse/
│   │   └── sse-writer.ts             # SSE format helper
│   ├── adapters/
│   │   └── adapter-registry.ts       # Maps adapter type -> command builder
│   └── types/
│       └── execution-types.ts        # Request/response types
├── Dockerfile
├── package.json
└── tsconfig.json
```

## Related Code Files

### Files to Create
All files above (~18 files).

## Implementation Steps

1. **Create `main.ts`**
   ```typescript
   import express from 'express';
   const app = express();
   app.use(express.json());
   app.post('/execute', executeRoute);
   app.post('/cancel', cancelRoute);
   app.get('/health', healthRoute);
   app.listen(8080, () => console.log('Agent Executor ready on :8080'));

   // Graceful shutdown
   process.on('SIGTERM', () => {
     // Kill all child processes
     activeProcesses.forEach(p => p.kill('SIGTERM'));
     setTimeout(() => process.exit(0), 10000);
   });
   ```

2. **Create `sse-writer.ts`**
   ```typescript
   export class SSEWriter {
     constructor(private res: Response) {
       res.setHeader('Content-Type', 'text/event-stream');
       res.setHeader('Cache-Control', 'no-cache');
       res.setHeader('Connection', 'keep-alive');
     }
     send(eventType: string, data: unknown) {
       this.res.write(`event: ${eventType}\ndata: ${JSON.stringify(data)}\n\n`);
     }
     close() { this.res.end(); }
   }
   ```

3. **Create `execution-pipeline.ts`**
   - Orchestrates steps 1-10 from blueprint 07
   - Accepts request + SSEWriter
   - Returns void (result sent via SSE)
   - Maintains reference to child process for cancellation

4. **Create pipeline steps** (one file each for modularity):

   **request-validator.ts**: Validate required fields, adapter type recognized

   **workspace-resolver.ts**:
   - If repoUrl provided and not cloned: `git clone <repoUrl> <cwd>`
   - If branch specified: `git checkout <branch>` or `git worktree add`
   - Ensure cwd exists: `mkdir -p`

   **environment-builder.ts**:
   - Start with process.env
   - Add AGENT_ID, COMPANY_ID, RUN_ID, API_URL, API_KEY
   - Map apiKeyProvider to env var: `{ anthropic: 'ANTHROPIC_API_KEY', openai: 'OPENAI_API_KEY', google: 'GOOGLE_API_KEY' }`
   - Add wake context: TASK_ID, WAKE_REASON, etc.
   - Add workspace vars
   - Add custom env from adapter config
   - Strip Claude session vars to prevent nested session errors

   **skill-linker.ts**:
   - For each skill in request.skills: symlink from `/app/skills/<name>/` to adapter-specific directory
   - Adapter-specific paths: `~/.claude/skills/`, `~/.codex/skills/`, etc.

   **command-builder.ts**:
   - Delegates to adapter registry for command + args
   - Claude: `claude --model <model> --output-format json [--effort high] [--context-file ...]`
   - Codex: `codex --model <model> [--search]`
   - Generic: returns `{ command, args }` tuple

   **prompt-renderer.ts**:
   - Template: "You are {role} at {company}. Your current task: {taskTitle}. Priority: {priority}. Follow task-protocol skill."
   - Piped to child stdin

   **process-spawner.ts**:
   - `child_process.spawn(command, args, { cwd, env, stdio: ['pipe', 'pipe', 'pipe'] })`
   - Write rendered prompt to stdin, then end stdin
   - Return child process reference

   **output-streamer.ts**:
   - `child.stdout.on('data', chunk => { seq++; sse.send('log', { stream: 'stdout', chunk: chunk.toString(), seq }); })`
   - Same for stderr
   - Also attempt to parse adapter-specific result JSON from stdout

   **timeout-enforcer.ts**:
   - `setTimeout(() => child.kill('SIGTERM'), timeoutMs)`
   - After 10s grace: `child.kill('SIGKILL')`

   **result-collector.ts**:
   - `child.on('exit', (code, signal) => { ... })`
   - Parse usage from output (adapter-specific)
   - Build result object: exitCode, signal, timedOut, usage, sessionParams, provider, model, summary
   - SSE send 'result' event, then close stream

5. **Create `adapter-registry.ts`**
   - Maps adapter type string to command builder function
   - Imports from `@ai-company/adapters` package (Phase 15)
   - Fallback: throw error for unknown adapter type

6. **Create routes**
   - `execute-route.ts`: Check no active execution (409 if busy), create SSEWriter, run pipeline
   - `cancel-route.ts`: Find active process by runId, send SIGTERM
   - `health-route.ts`: Return `{ status: 'ok', uptime, activeRuns }`

7. **Create `Dockerfile`**
   ```dockerfile
   FROM node:20-slim AS base
   RUN npm install -g pnpm

   FROM base AS installer
   WORKDIR /app
   COPY pnpm-workspace.yaml package.json turbo.json ./
   COPY packages/shared/ ./packages/shared/
   COPY packages/adapter-utils/ ./packages/adapter-utils/
   COPY packages/adapters/ ./packages/adapters/
   COPY apps/executor/ ./apps/executor/
   RUN pnpm install --frozen-lockfile

   FROM installer AS builder
   RUN pnpm --filter @ai-company/executor build

   FROM node:20-slim AS runner
   WORKDIR /app
   RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
   RUN npm install -g @anthropic-ai/claude-code
   COPY --from=builder /app/apps/executor/dist ./dist
   COPY --from=builder /app/node_modules ./node_modules
   COPY config/skills/ ./skills/

   EXPOSE 8080
   CMD ["node", "dist/main.js"]
   ```

## Todo List
- [ ] Express server + routes
- [ ] SSE writer utility
- [ ] Execution pipeline orchestrator
- [ ] 10 pipeline step modules
- [ ] Adapter registry
- [ ] Concurrency guard (409 on busy)
- [ ] Graceful shutdown handler
- [ ] Dockerfile
- [ ] typecheck passes
- [ ] Manual test with mock adapter

## Success Criteria
- POST /execute streams SSE events and final result
- 409 returned on concurrent execution attempt
- SIGTERM/SIGKILL correctly enforced on timeout
- Child processes cleaned up on shutdown
- Dockerfile builds successfully

## Risk Assessment
- **Agent CLI not installed:** Dockerfile must pre-install all CLIs. Test image build.
- **Workspace persistence:** Fly.io volume must be mounted at /workspace
- **Memory leaks:** SSE streams must be properly closed on client disconnect

## Security Considerations
- API keys exist only in child process env vars, never on disk
- Scrub API keys from stdout/stderr before streaming
- Validate runId matches expected format
- No unauthenticated access (executor only reachable from control plane internal network)

## Next Steps
- Phase 15: Adapter implementations used by command-builder
