# 07 — Agent Executor Specification

The Agent Executor is a lightweight Node.js process that runs inside each Fly.io VM. It receives execution requests from the control plane, spawns agent processes, and streams results back.

## Overview

```
Control Plane                              Fly.io VM
     │                                        │
     │  POST /execute ─────────────────────► Agent Executor (:8080)
     │                                        │
     │                                        ├─ Resolve adapter
     │                                        ├─ Build environment
     │                                        ├─ Symlink skills
     │                                        ├─ Spawn agent process
     │                                        │    (claude, codex, etc.)
     │                                        │
     │  ◄──── SSE stream (events) ───────────┤
     │                                        │
     │  ◄──── Final result JSON ─────────────┤
     │                                        │
     │  GET /health ─────────────────────────► 200 OK
     │                                        │
     │  POST /cancel ────────────────────────► SIGTERM → SIGKILL
```

## HTTP API

### POST /execute

Receives an execution request from the control plane's Execution Engine.

**Request:**
```json
{
  "runId": "uuid",
  "agentId": "uuid",
  "companyId": "uuid",
  "adapter": {
    "type": "claude",
    "config": {
      "model": "claude-sonnet-4-20250514",
      "effort": "high",
      "timeoutSec": 600,
      "env": {
        "NODE_ENV": "production"
      }
    }
  },
  "apiKey": "sk-ant-...",
  "apiKeyProvider": "anthropic",
  "workspace": {
    "cwd": "/workspace/project",
    "repoUrl": "https://github.com/org/repo.git",
    "repoRef": "main",
    "branch": "feature/APP-42"
  },
  "context": {
    "taskId": "uuid",
    "wakeReason": "assignment",
    "commentId": null,
    "approvalId": null,
    "approvalStatus": null,
    "linkedIssueIds": []
  },
  "session": {
    "sessionId": "prev-session-id",
    "cwd": "/workspace/project",
    "workspaceId": "uuid"
  },
  "authToken": "<agent-jwt-48h>",
  "skills": ["task-protocol"]
}
```

**Response:** SSE stream (see below), then final JSON result.

### GET /health

```json
{ "status": "ok", "uptime": 1234, "activeRuns": 0 }
```

### POST /cancel

```json
{ "runId": "uuid" }
```

Sends SIGTERM to the running agent process. After 10-second grace period, sends SIGKILL.

## SSE Event Stream Format

The `/execute` endpoint responds with `Content-Type: text/event-stream`. Each event:

```
event: <event_type>
data: <json_payload>

```

### Event Types

**`status`** — Run status change
```json
{ "status": "running", "timestamp": "2026-03-16T14:00:00Z" }
```

**`log`** — Agent stdout/stderr output
```json
{
  "stream": "stdout",
  "chunk": "Reading issue context...\n",
  "seq": 1
}
```
```json
{
  "stream": "stderr",
  "chunk": "Warning: rate limit approaching\n",
  "seq": 2
}
```

**`system`** — Executor system messages
```json
{
  "level": "info",
  "message": "Spawning claude with model claude-sonnet-4-20250514",
  "details": {
    "command": "claude",
    "cwd": "/workspace/project",
    "adapter": "claude"
  }
}
```

**`result`** — Final execution result (last event)
```json
{
  "exitCode": 0,
  "signal": null,
  "timedOut": false,
  "usage": {
    "inputTokens": 85000,
    "outputTokens": 12000,
    "cachedInputTokens": 20000,
    "costUsd": 0.44
  },
  "sessionParams": {
    "sessionId": "new-session-id",
    "cwd": "/workspace/project",
    "workspaceId": "uuid"
  },
  "sessionDisplayId": "new-session-id",
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",
  "billingType": "api",
  "summary": "Completed login page implementation"
}
```

**`error`** — Execution error
```json
{
  "code": "ADAPTER_SPAWN_FAILED",
  "message": "claude: command not found",
  "details": null
}
```

## Execution Pipeline (Inside Agent Executor)

```
1. VALIDATE REQUEST
   - Required fields present
   - Adapter type recognized
   - Workspace directory exists (or create it)

2. RESOLVE WORKSPACE
   IF workspace.repoUrl AND repo not cloned:
     git clone <repoUrl> <cwd>
   IF workspace.branch:
     git checkout <branch> OR git worktree add <path> <branch>
   Ensure cwd exists (mkdir -p)

3. BUILD ENVIRONMENT
   Start with process.env, then merge:
   {
     AGENT_ID: request.agentId,
     COMPANY_ID: request.companyId,
     RUN_ID: request.runId,
     API_URL: process.env.CONTROL_PLANE_URL,
     API_KEY: request.authToken,

     // API key injection — explicit mapping from provider to env var
     const apiKeyEnvMap = {
       anthropic: 'ANTHROPIC_API_KEY',
       openai: 'OPENAI_API_KEY',
       google: 'GOOGLE_API_KEY',
     };
     env[apiKeyEnvMap[request.apiKeyProvider]] = request.apiKey;

     // Wake context
     TASK_ID: request.context.taskId,
     WAKE_REASON: request.context.wakeReason,
     WAKE_COMMENT_ID: request.context.commentId,
     APPROVAL_ID: request.context.approvalId,
     APPROVAL_STATUS: request.context.approvalStatus,
     LINKED_ISSUE_IDS: request.context.linkedIssueIds?.join(","),

     // Workspace
     WORKSPACE_CWD: request.workspace.cwd,
     WORKSPACE_REPO_URL: request.workspace.repoUrl,
     WORKSPACE_BRANCH: request.workspace.branch,

     // Custom env from adapter config
     ...request.adapter.config.env
   }

   IMPORTANT: Strip any nesting vars that could confuse agent CLIs
   (e.g., CLAUDE_CODE_SESSION vars if already inside Claude)

4. SETUP SKILLS
   For each skill in request.skills:
     Determine agent CLI skill directory:
       claude  → ~/.claude/skills/<skill-name>/
       codex   → ~/.codex/skills/<skill-name>/
       cursor  → ~/.cursor/skills/<skill-name>/
       gemini  → ~/.gemini/skills/<skill-name>/
       opencode→ ~/.opencode/skills/<skill-name>/
       pi      → ~/.pi/skills/<skill-name>/
     Symlink: /app/skills/<skill-name>/ → target directory
     Ensure symlink is valid (remove stale, recreate)

5. BUILD COMMAND + ARGS (adapter-specific, see adapter guide)
   Example for Claude:
     command: "claude"
     args: ["--model", "claude-sonnet-4-20250514", "--output-format", "json"]
     If session exists: args.push("--context-file", sessionFilePath)
     If effort set: args.push("--effort", effort)

6. RENDER STDIN PROMPT
   Template:
     "You are {agent.role} at {company.name}.
      Your current task: {context.taskTitle}
      Priority: {context.taskPriority}

      Follow the task-protocol skill for all platform interactions."

   Pipe to stdin of spawned process.

7. SPAWN CHILD PROCESS
   child = spawn(command, args, {
     cwd: workspace.cwd,
     env: mergedEnv,
     stdio: ['pipe', 'pipe', 'pipe'],
     timeout: adapter.config.timeoutSec * 1000
   })

   child.stdin.write(renderedPrompt)
   child.stdin.end()

8. STREAM OUTPUT
   child.stdout.on('data', (chunk) => {
     seq++
     emit SSE: event=log, data={ stream: "stdout", chunk, seq }
     // Also: parse for adapter-specific result JSON (see adapter guide)
   })

   child.stderr.on('data', (chunk) => {
     seq++
     emit SSE: event=log, data={ stream: "stderr", chunk, seq }
   })

9. ENFORCE TIMEOUT
   setTimeout(() => {
     child.kill('SIGTERM')
     setTimeout(() => {
       if (!child.killed) child.kill('SIGKILL')
     }, 10000)  // 10-second grace period
   }, adapter.config.timeoutSec * 1000)

10. COLLECT RESULT
    child.on('exit', (code, signal) => {
      result = {
        exitCode: code,
        signal: signal,
        timedOut: (signal === 'SIGTERM' || signal === 'SIGKILL'),
        usage: parseUsageFromOutput(stdout),
        sessionParams: parseSessionFromOutput(stdout),
        provider: detectProvider(adapter.type),
        model: adapter.config.model,
        summary: parseSummaryFromOutput(stdout)
      }
      emit SSE: event=result, data=result
      close SSE stream
    })
```

## Process Signal Handling

```
Normal exit:     code 0              → status: "succeeded"
Error exit:      code != 0           → status: "failed"
Timeout:         SIGTERM after N sec → wait 10s → SIGKILL → status: "timed_out"
Cancel request:  SIGTERM             → wait 10s → SIGKILL → status: "cancelled"
Crash:           unexpected signal   → status: "failed", error: signal name
```

## Concurrency

V1: **One execution at a time per VM.** If a second request arrives while one is running, return `409 Conflict`.

Future: Support concurrent executions (different agents sharing the VM).

## Docker Image

```dockerfile
FROM node:lts-slim

# Pre-install agent CLIs
RUN npm install -g @anthropic-ai/claude-code @openai/codex opencode

# Install git (for workspace cloning)
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*

# Copy executor code
WORKDIR /app
COPY agent-executor/ ./
COPY skills/ ./skills/

RUN npm install

EXPOSE 8080
CMD ["node", "index.js"]
```

## Startup

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/execute', handleExecute);
app.post('/cancel', handleCancel);
app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.listen(8080, () => console.log('Agent Executor ready on :8080'));
```

## Key Implementation Notes

1. **Never store API keys to disk.** They exist only as env vars in the child process.
2. **Always clean up child processes.** On executor shutdown, SIGTERM all children.
3. **Log redaction.** Scrub API keys and home paths from all streamed output.
4. **Workspace persistence.** `/workspace` is on a Fly.io persistent volume — git repos survive VM hibernation.
5. **Skill symlinks.** Create before spawn, clean up after (or leave for next run).
6. **Sequence numbers.** Every SSE event gets monotonically increasing `seq` for ordered replay.
