# 00 — Project Overview

## What This Project Does

A web platform that lets anyone create and run a **fully autonomous AI company**. Users define a business goal, the platform assembles a team of AI agents, and those agents work 24/7 — writing code, designing UI, running marketing campaigns, managing projects — all without human intervention.

The user sees a dashboard. Underneath, a fleet of AI agents coordinate through an org chart, check out tasks, do work, delegate to each other, and report results.

## What Problem It Solves

Starting a business requires hiring people, managing teams, paying salaries, renting offices. A 15-person startup costs $250,000+/month. Most entrepreneurs can't afford that.

This platform replaces the entire team with AI agents for ~$2,000/month. Same output. 140x cheaper. Works 24/7. Scales instantly.

## Target Users

**Primary:** Non-technical entrepreneurs who have business ideas but can't afford (or don't want) human teams.

**Not for:** People who want a chatbot or a single AI assistant. This is for running companies, not having conversations.

## Key Capabilities

| Capability | What It Means for Users |
|-----------|------------------------|
| AI Team Assembly | "Hire" a CEO, engineers, designers, marketers from templates |
| Autonomous Execution | Agents work independently, delegate, collaborate |
| Goal Alignment | Every task traces back to company mission |
| Cost Control | Budget caps per agent, spending alerts, kill switch |
| Governance | Approve hires, review strategy, override decisions |
| Real-time Dashboard | Watch your company work in real-time |
| Self-Growing Teams | Agents can hire other agents (with approval) |
| API Key Based | Users bring their own LLM API keys (Anthropic, OpenAI, etc.) |

## How AI Fits In

The platform is a **control plane** — it doesn't build AI models or write prompts. It orchestrates existing AI agents:

1. **Schedules** agent execution via heartbeats (discrete work windows)
2. **Provides context** — goals, tasks, workspace, conversation history
3. **Coordinates** — atomic task checkout prevents double-work
4. **Tracks** — costs, logs, tool calls, outcomes
5. **Governs** — budgets, approvals, pause/terminate

Agents bring their own intelligence (Claude, GPT, Gemini, etc.). The platform provides the organizational structure.

## Major System Parts

```
Platform
├── Web App (React)
│   ├── Dashboard — company metrics at a glance
│   ├── Team Management — hire, configure, monitor agents
│   ├── Task Board — issues, assignments, progress
│   ├── Org Chart — hierarchical team visualization
│   ├── Approvals — governance workflow
│   └── Billing — usage, credits, API key management
│
├── Control Plane (Node.js API Server)
│   ├── Heartbeat Scheduler — periodic agent execution
│   ├── Execution Engine — cloud-based agent execution via Fly.io
│   ├── Adapter Registry — 9+ agent runtime integrations
│   ├── Task Coordination — atomic checkout, delegation
│   ├── Cost Tracking — token + compute usage
│   └── Real-time Events — WebSocket live updates
│
├── Execution Plane (Fly.io VMs)
│   ├── Per-company VM — agents run on isolated Fly.io Machines
│   ├── Agent Executor — receives and runs execution requests
│   ├── Agent processes — Claude, Codex, OpenClaw, etc.
│   └── Persistent workspace — git repos, files
│
├── Database (PostgreSQL via Neon/Supabase, multi-tenant)
│   └── 35+ tables — companies, agents, issues, runs, costs, etc.
│
└── Skills (Agent Instructions)
    └── Agents discover task protocol at runtime
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React, Vite, Tailwind CSS, shadcn/ui, React Query |
| Backend | Node.js, NestJS, TypeScript, @nestjs/schedule |
| Database | PostgreSQL (Neon/Supabase managed) |
| Auth | Better Auth (sessions), Agent JWT, encrypted API keys |
| Execution | Fly.io Machines (per-company VMs) |
| Storage | S3 |
| Real-time | Redis pub/sub (Upstash) + WebSocket |
| Monorepo | pnpm workspaces |
| Testing | Vitest, Playwright |
| CI/CD | GitHub Actions |

## Deployment Model

This is a **cloud-only** platform. All agent execution runs on Fly.io VMs. Users access the platform via a hosted web dashboard. There is no local or self-hosted execution mode.

| Component | Provider | Notes |
|-----------|----------|-------|
| API Server | Railway or Fly.io | Always cloud-hosted |
| Database | Neon / Supabase | Managed PostgreSQL |
| Redis | Upstash | Pub/sub events |
| Storage | AWS S3 | Files, logs |
| Agent Execution | Fly.io Machines | Per-company VMs |
| LLM API Calls | User's own keys | Anthropic, OpenAI, Google |
