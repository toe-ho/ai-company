# Project Overview & Product Development Requirements

## Vision Statement

Enable non-technical entrepreneurs to launch and operate AI-powered businesses for ~$2,000/month instead of hiring a 15-person team for $250,000+/month. Replace human teams entirely with autonomous AI agents that work 24/7 without human intervention.

## Target Users

**Primary:** Non-technical entrepreneurs with viable business ideas who lack capital or desire for traditional hiring.

**Secondary:** Researchers, product teams experimenting with AI-driven workflows.

**Not For:** Chatbot users, single-assistant seekers, or companies wanting to augment existing teams.

## Business Model

| Component | Details |
|-----------|---------|
| **Platform Fee** | $49-99/month per user account |
| **Compute Costs** | Markup on Fly.io Machines (20-30% margin) |
| **LLM Costs** | Users pay directly to Anthropic, OpenAI, Google (platform transparent) |
| **Revenue Model** | Platform subscription + compute margin |

## Core Capabilities

1. **AI Team Assembly** — Create company from templates, hire agents (CEO, engineers, designers, marketers)
2. **Autonomous Execution** — Agents work independently, delegate to each other, coordinate
3. **Goal Alignment** — Every task traces back to company mission and OKRs
4. **Cost Control** — Per-agent budgets, monthly caps, spending alerts, kill switch
5. **Governance** — Approve hires, review strategy, override decisions
6. **Real-time Visibility** — Watch agents work via dashboard, see live execution logs
7. **Self-Growing Teams** — Agents can propose new hires (with human approval)
8. **User-Owned Keys** — Users provide their own LLM API keys (no key vendor lock-in)

## Domain Concepts

### Company
A virtual organization created by a user. Contains agents, projects, tasks, budget. One user can own multiple companies.

### Agent
An AI worker with a role (CEO, engineer, designer, etc.). Executes tasks in discrete heartbeat windows. Has budget, status, org hierarchy (reports-to).

### Issue (Task)
A unit of work assigned to an agent or company. Has title, description, status, priority. Can spawn subtasks. Only one agent can checkout issue at a time.

### Heartbeat
A discrete execution window (e.g., every 30 minutes). Agent receives context (tasks, goals, history), makes decisions, takes actions. Returns immediately (no streaming).

### Execution Plane
Fly.io VM per company. Runs agent executor process. Agents run inside this VM. Isolated, persistent workspace (git repos, files).

### Control Plane
NestJS API server. Orchestrates heartbeats, tracks costs, manages permissions, stores company data. Single instance serves all users.

### Adapter
Integration for a specific LLM provider (Claude, GPT-4, Gemini, etc.). Translates execution requests to provider API calls. Handles streaming, parsing responses.

### Skill
SKILL.md file containing agent instructions. Defines what agent can do, what tools available, how to work. Discovered and read by agent at runtime.

### Cost Event
Record of compute/token usage. Per-heartbeat, per-agent. Tracked for billing and budget enforcement.

## Functional Requirements

### User Management
- [ ] Sign up / login (email)
- [ ] Multi-company support (own multiple companies)
- [ ] Company invitation / team member roles (owner, admin, viewer)
- [ ] LLM API key vault (encrypt, decrypt, validate)
- [ ] Billing & subscription management

### Company Setup
- [ ] Create company from template or blank
- [ ] Configure company metadata (name, description, mission)
- [ ] Set monthly budget
- [ ] Select Fly.io region for VM

### Agent Management
- [ ] Hire agent from predefined roles (CEO, engineer, designer, etc.)
- [ ] Configure agent (model, heartbeat interval, budget, permissions)
- [ ] Organize agents in hierarchy (reports-to relationships)
- [ ] Pause / resume / terminate agents
- [ ] View agent status and runtime metrics

### Task Management
- [ ] Create issue (title, description, priority, assignment)
- [ ] Atomic checkout (only one agent at a time)
- [ ] Update status (backlog → in progress → done)
- [ ] Comments and discussion
- [ ] Attachments (files, links)
- [ ] Labels and filtering

### Execution & Monitoring
- [ ] Trigger heartbeat manually
- [ ] View live execution logs (stdout/stderr)
- [ ] Stream execution events to dashboard in real-time
- [ ] View heartbeat history and results
- [ ] Cancel running execution
- [ ] Retry failed heartbeat

### Governance & Approvals
- [ ] Approval workflow for new hires (before agent created)
- [ ] Approval workflow for budget increases
- [ ] View all agent actions in activity log
- [ ] Org chart visualization
- [ ] Strategy review (goals vs actual execution)

### Cost Tracking & Budgets
- [ ] Per-agent monthly budget
- [ ] Per-company monthly budget
- [ ] Cost breakdown by agent, model, provider
- [ ] Budget alerts (80%, 100%)
- [ ] Hard stop at monthly cap (no execution beyond)
- [ ] Export cost reports

### Dashboard & Reporting
- [ ] Company overview (health, metrics at a glance)
- [ ] Agent status (active, idle, error count)
- [ ] Recent tasks and progress
- [ ] Cost summary and trends
- [ ] Team org chart
- [ ] Goal progress tracking

## Non-Functional Requirements

### Performance
- API response time <200ms (p95)
- Real-time dashboard updates <1s latency
- Heartbeat start-to-finish <5 minutes (typical)

### Reliability
- 99.9% uptime for API server
- Database backups (automated)
- Graceful degradation on external API failures (LLM providers)
- Orphan run cleanup (5+ min without update)

### Security
- HTTPS only
- Session cookies for users (Better Auth)
- JWT for agent callbacks
- AES-256 encryption for LLM API keys at rest
- Multi-tenant row-level isolation (no data leaks)
- Rate limiting on public endpoints
- Input validation (Zod on all DTOs)

### Scalability
- Horizontal scaling of API server (stateless)
- Per-company VM isolation (1 VM = 1 company)
- Database read replicas (Neon Pro)
- Redis cluster ready (Upstash Pro)
- Concurrent agent execution (multiple agents in parallel)

### Observability
- Structured logging (Pino)
- Request/response logging
- Execution event streaming
- Cost tracking per heartbeat
- Activity log (audit trail)

## Design Principles

### 1. User-Centric
Make it trivial for non-technical users. Hide complexity. Defaults over configuration.

### 2. Cost Transparency
Always show users what they're paying for. No surprise charges.

### 3. Fail Safe
If anything breaks, don't silently fail. Alert user. Provide manual override.

### 4. Isolation
One company's agents never see another's data. One agent's failure never cascades.

### 5. Autonomy First
Agents should work independently by default. Humans govern, not micromanage.

### 6. Heartbeat Model
Discrete windows prevent runaway costs and enable human oversight. No continuous execution.

### 7. Clean Architecture
Domain layer never imports framework code. Clear separation of concerns (4 layers).

### 8. Multi-tenant by Default
Every entity has companyId. Every query filters by it. No exceptions.

## Success Metrics

| Metric | Target |
|--------|--------|
| **Time to First Heartbeat** | <10 minutes from signup |
| **Agent Autonomy** | 80%+ of tasks completed without human intervention |
| **Platform Uptime** | 99.9% |
| **Cost Predictability** | Actual vs budgeted within 5% |
| **User Adoption** | 1K+ active companies in Year 1 |
| **Revenue per User** | $500-1000/year (platform + compute margin) |

## Technical Constraints

1. **No local execution** — All agent execution on Fly.io VMs (cloud-only)
2. **No custom models** — Agents use existing provider APIs (Claude, GPT, etc.)
3. **No self-hosted option** — SaaS-only (managed cloud platform)
4. **No agent persistence across runs** — Sessions serialized between heartbeats
5. **No real-time execution** — Heartbeat-based (discrete windows, not streaming)
6. **No agent communication outside protocol** — All inter-agent communication via task system

## Implementation Phases

### Phase 1: Core Platform (Weeks 1-4)
Monorepo, database, auth, basic CQRS structure. Manual heartbeat triggers. No agent execution yet.

### Phase 2: AI Execution (Weeks 5-8)
Adapters, agent executor on Fly.io VM, heartbeat scheduler, execution engine. Live execution with real agents.

### Phase 3: Frontend (Weeks 9-12)
React dashboard, team management, task board, org chart. Real-time events via WebSocket.

### Phase 4: Advanced Features (Weeks 13+)
Company templates, approvals, cost analytics, self-hiring agents, integrations.

## Dependencies

| Dependency | Purpose | Notes |
|-----------|---------|-------|
| **PostgreSQL** | Multi-tenant database | Neon or Supabase managed |
| **Redis** | Real-time pub/sub | Upstash for simplicity |
| **Fly.io** | Per-company VM execution | Machines API for lifecycle |
| **AWS S3** | Logs and attachments | Minimal usage (~$1-5/mo) |
| **Better Auth** | Session authentication | Open source, self-hosted logic |

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **LLM API outage** | Agents can't execute | Implement fallback, notify user |
| **Fly.io VM cost explosion** | Margin eaten by compute | Per-agent CPU/time limits, heartbeat throttle |
| **Multi-tenant data leak** | Regulatory nightmare | Row-level isolation, encryption at rest, audit logs |
| **Runaway agent** | Uncontrolled spending | Hard budget caps, timeout limits, orphan reaping |
| **Database scaling** | Query slowness | Read replicas (Neon Pro), query optimization, indexes |

## Acceptance Criteria

### MVP (Minimum Viable Product)
- [ ] Users can sign up and create company
- [ ] Hire 3+ agents from template
- [ ] Manual heartbeat execution (trigger via dashboard)
- [ ] View heartbeat logs and results
- [ ] Cost tracking and budget caps
- [ ] Real-time log streaming to dashboard

### Phase 1 Complete
- [ ] All Phase 1 requirements done
- [ ] Scheduled heartbeats (automatic every N minutes)
- [ ] Agent-to-agent delegation
- [ ] Issue tracking with task checkout
- [ ] Org chart visualization

### Phase 2 Complete
- [ ] 9+ adapters working (Claude, GPT, Gemini, etc.)
- [ ] Agent executor running on Fly.io
- [ ] Agents writing code, running tools
- [ ] Cost tracking per tool invocation
- [ ] Session persistence across heartbeats

### Phase 3 Complete
- [ ] Full React dashboard
- [ ] Team management UI
- [ ] Real-time WebSocket updates
- [ ] Mobile responsive
- [ ] Accessibility (WCAG AA)

### Phase 4+ Complete
- [ ] Company templates
- [ ] Approval workflows
- [ ] Cost analytics & budgets
- [ ] Self-hiring agents
- [ ] Integrations (GitHub, Slack, etc.)
