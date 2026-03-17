# 01 — Product Concept

## The Product Idea

A platform where non-technical users launch AI-powered companies. User provides a goal ("Build an AI note-taking app"), the platform creates an org chart of AI agents, and those agents autonomously build the product — writing code, designing interfaces, running marketing, managing tasks.

**What users see:** A project management dashboard with an AI team working for them.
**What happens underneath:** AI agents coordinate via heartbeats, check out tasks atomically, use LLM APIs to do work, and report back.

## User Problems Being Solved

| Problem | Solution |
|---------|---------|
| Can't afford a dev team ($250K+/month) | AI team for ~$2K/month |
| Don't know how to code | AI agents handle all technical work |
| Can't manage 20 terminals of AI tools | One dashboard, full visibility |
| Afraid of runaway AI costs | Budget caps, spending alerts, kill switch |
| Want business running 24/7 | Agents work around the clock |
| Don't understand AI configuration | Template-based setup, zero config |

## Core User Journeys

### Journey 1: Launch a Company
```
1. Sign up → Dashboard
2. Click "Create Company"
3. Choose template: "AI SaaS Startup" / "Content Agency" / "Dev Shop"
4. Describe your goal: "Build a task management app"
5. Platform suggests team: CEO, 2 Engineers, Designer, QA
6. User enters API key (Anthropic/OpenAI) — guided with screenshots
7. Click "Launch"
8. Watch agents start working on the dashboard
```

### Journey 2: Monitor and Govern
```
1. Dashboard shows: active agents, tasks in progress, costs
2. CEO agent posts strategy update → user reviews
3. CEO requests to hire a Marketing agent → approval notification
4. User approves → platform provisions agent on Fly.io VM
5. Marketing agent starts creating content
6. User sees costs climbing → adjusts budget caps
7. Agent completes task → user reviews output
```

## Main User Actions

- Create company from template
- Enter API key (guided setup)
- Watch dashboard, review agent output
- Approve/reject agent requests (hires, strategy)
- Adjust budgets, pause/terminate agents
- Read summaries (not raw logs)

## Product Value Proposition

> "Describe your business idea. We'll build the team. They'll build the business."

## Business Model

### Revenue Streams
1. **Platform fee:** $49-99/month (access to cloud dashboard)
2. **Compute markup:** User pays Fly.io cost + margin (~$20-50/company)
3. **Future: LLM proxy:** Buy API tokens at volume, resell with markup
4. **Future: Template marketplace:** Pre-built company templates

### User Pays Directly
- LLM API tokens (to Anthropic/OpenAI/Google) — via their own API key
- Platform fee (to us)

### We Do NOT Handle (Phase 1-2)
- LLM billing (user's own API key)
- Subscription management for LLM providers
- Model fine-tuning or hosting

## Important Domain Concepts

### Company
The organizational unit. Has goals, budget, team of agents, task hierarchy. Each user can run multiple companies. Full data isolation between companies.

### Agent
An AI worker. Has: role (CEO, engineer, etc.), adapter type (which AI runtime), budget, permissions, reporting hierarchy. Does NOT run continuously — works in discrete heartbeats.

### Heartbeat
A discrete execution window. Agent wakes, checks assignments, does work, exits. Prevents runaway costs. Agents resume context on next heartbeat.

### Issue (Task)
The unit of work. Has status (backlog → todo → in_progress → done), single assignee, parent hierarchy. Only one agent can work on a task at a time (atomic checkout).

### Execution Engine
WHERE and HOW the agent executes. All agents run on Fly.io VMs in the cloud. The Execution Engine is transparent to the agent — it receives standard env vars and calls the same control plane API regardless of which VM it runs on.

### Adapter
HOW the agent communicates with a specific AI runtime. Claude adapter spawns `claude` CLI. Codex adapter spawns `codex` CLI. Adapters run inside the Fly.io VM and are independent of the underlying VM infrastructure.

### Template
Pre-configured company setup. Includes: org chart, agent roles, adapter types, budget defaults, initial goals. User picks a template, customizes the goal, and launches.

## Design Principles

1. **Non-tech first** — No terminal, no YAML, no config. Wizards and templates
2. **Progressive disclosure** — Simple view by default, "Advanced" for power users
3. **Output-first** — Summaries, not raw logs. Work isn't done until user sees result
4. **Safe autonomy** — Agents work independently but humans approve big decisions
5. **API key only** — No subscription management. User enters API key, we handle the rest
6. **Cost transparency** — User always knows what they're spending and why
