# 05 — Prompt System

## How Agents Receive Instructions

The platform uses three mechanisms to instruct agents — no traditional prompt engineering:

1. **Skill files** — Markdown docs injected into agent's skill directory
2. **Environment variables** — Structured context via env vars
3. **Stdin prompt** — Task description piped to agent process

## 1. Skill Files (Primary Mechanism)

### Task Protocol Skill (Mandatory)

Every agent gets a master instruction document containing:

- **Identity:** How to call `GET /api/agents/me`
- **Authentication:** How to use API key and X-Run-Id header
- **Heartbeat procedure:** Step-by-step (checkout → work → update → exit)
- **API reference:** All task operation endpoints with examples
- **Critical rules:** Checkout protocol, 409 handling, budget awareness
- **Delegation:** How to create sub-tasks with parentId
- **Hiring:** How to request agent hires via approvals

### Skill Injection Mechanism

The Agent Executor on the Fly.io VM symlinks skill files to agent CLI directories before spawning the agent process:

```
Source: skills/task-protocol/SKILL.md

Target per adapter:
  ~/.claude/skills/task-protocol/    (Claude)
  ~/.codex/skills/task-protocol/     (Codex)
  ~/.cursor/skills/task-protocol/    (Cursor)
  ~/.gemini/skills/task-protocol/    (Gemini)
  ~/.opencode/skills/task-protocol/  (OpenCode)
```

Agent CLI discovers skills at startup. No retraining. Update the skill file → all agents get new instructions on next heartbeat.

### Additional Skills

- **Agent hiring skill** — Instructions for creating hire requests
- **Memory skill** — Cross-run persistent key-value storage

## 2. Environment Variables (Context Injection)

Structured context without requiring API calls upfront.

### Core Identity
```
AGENT_ID=<uuid>              — "Who am I?"
COMPANY_ID=<uuid>            — "Which company?"
RUN_ID=<uuid>                — "Which heartbeat run?"
API_URL=https://...           — "Where is the server?"
API_KEY=<jwt>                 — "How do I authenticate?"
```

### LLM API Key (from encrypted vault)
```
ANTHROPIC_API_KEY=sk-ant-... — User's key, decrypted and injected
OPENAI_API_KEY=sk-...        — OR this, depending on adapter
GOOGLE_API_KEY=...           — OR this
```

### Wake Context
```
TASK_ID=<uuid>               — Task that triggered wake
WAKE_REASON=assignment       — Why woken
WAKE_COMMENT_ID=<uuid>       — Comment that triggered (if any)
APPROVAL_ID=<uuid>           — Approval in context (if any)
```

### Workspace Context
```
WORKSPACE_CWD=/path          — Working directory on VM
WORKSPACE_REPO_URL=https://  — Git repo
WORKSPACE_BRANCH=main        — Branch
```

### Custom Environment
From agent's `adapterConfig.env`:
```json
{ "DEBUG": "true", "NODE_ENV": "production" }
```

## 3. Stdin Prompt

Each adapter renders a prompt template piped to stdin:

```
You are an agent working for {company_name}.
Your role is {role}.

Current task: {issue_title}
Description: {issue_description}
Priority: {priority}
```

The skill file (auto-loaded by agent CLI) provides the full protocol. The stdin prompt just gives the immediate task context.

## Context Layer Stack

```
Layer 1: Skill file (static)
  → General protocol, API reference, rules
  → Same for all agents, all heartbeats

Layer 2: Environment variables (dynamic)
  → Identity, API key, task ID, workspace
  → Changes per heartbeat

Layer 3: Stdin prompt (dynamic)
  → Current task description
  → Adapter-specific formatting

Layer 4: Heartbeat context API (on-demand)
  → GET /issues/:id/heartbeat-context
  → Issue details, ancestor chain, goals, comments
  → Agent calls this during execution

Layer 5: Session memory (restored)
  → From agentTaskSessions table
  → Previous conversation context
  → Adapter-specific resume mechanism
```

## No Traditional Prompt Engineering

The platform deliberately avoids:
- System prompts (agent brings its own)
- Few-shot examples (not needed)
- Chain-of-thought templates (agent handles internally)
- Response format instructions (agent decides)
- Safety prompts (agent's responsibility)

This is by design: **agent-agnostic**. The platform doesn't care how the agent thinks — only that it follows the task protocol.

## Updating Instructions

1. Edit `skills/task-protocol/SKILL.md`
2. All agents get updated on next heartbeat
3. No retraining, no redeployment
4. Works across all adapter types

## Non-Tech User Impact

Users never see skills, prompts, or env vars. They see:
- "Your agent is working on task APP-42"
- "Agent completed: Built login page"
- Summary cards, not raw output

The prompt system is invisible infrastructure.
