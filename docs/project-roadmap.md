# Project Roadmap

**Status:** Blueprint phase. Timeline and phases are planned estimates subject to change.

## Overview

Multi-phase development path to build and launch the AI Company Platform. Focus on core functionality first, advanced features after MVP.

## Phase Breakdown

### Phase 1: Core Platform (Weeks 1-4)
**Focus:** Infrastructure, database, authentication, basic CQRS structure
**Status:** Not started
**Effort:** 160-200 hours

#### Deliverables
- [ ] Monorepo setup (Turborepo + pnpm workspaces)
- [ ] PostgreSQL database schema (Drizzle ORM)
  - users, companies, agents, issues, heartbeat_runs, cost_events, activity_log
  - Indexes and foreign keys
  - Migration scripts
- [ ] Better Auth integration (user signup/login)
- [ ] API server skeleton (NestJS, Express)
  - Project structure (domain, infrastructure, application, api layers)
  - Module setup (SharedModule, ApiModule, etc.)
- [ ] Core CQRS handlers
  - CreateCompanyCommand/Handler
  - CreateAgentCommand/Handler
  - CheckoutIssueCommand/Handler
- [ ] Repository pattern (Company, Agent, Issue repositories)
- [ ] Basic API endpoints (/api/companies, /api/agents, /api/issues)
- [ ] Zod validation on all DTOs
- [ ] Exception filter and guards
- [ ] TypeScript compilation and linting

#### Success Criteria
- [ ] TypeScript builds without errors
- [ ] All API endpoints return correct HTTP status codes
- [ ] Database migrations run successfully
- [ ] Unit tests cover core handlers (>80% coverage)
- [ ] API documentation (Swagger/OpenAPI)

#### Dependencies
- PostgreSQL (local or Neon)
- Node.js 20+
- pnpm

#### Risks
- Database schema changes late in development
- **Mitigation:** Careful design review before implementation
- Performance issues with multi-tenant queries
- **Mitigation:** Add indexes early, test with sample data

---

### Phase 2: AI Execution (Weeks 5-8)
**Focus:** Adapter implementations, agent executor on Fly.io, heartbeat scheduler
**Status:** Blocked on Phase 1
**Effort:** 200-240 hours

#### Deliverables
- [ ] Adapter registry and interface definitions
  - IAdapter interface (command, arguments, output parsing)
  - Per-adapter specifications (Claude, GPT, Gemini, Codex, Openclaw, etc.)
- [ ] Adapter implementations (9+ adapters)
  - Claude adapter (Anthropic API)
  - GPT adapter (OpenAI API)
  - Gemini adapter (Google API)
  - Codex, Openclaw, Cursor, Pi, Process, HTTP adapters
  - Session codec for state persistence
- [ ] Agent executor on Fly.io VM
  - HTTP server (POST /execute)
  - Subprocess management (spawn agent processes)
  - Workspace management (git, files)
  - SSE response streaming
- [ ] ExecutionEngineService
  - HTTP POST to Fly.io VM
  - SSE event parsing (stdout, stderr, done)
- [ ] ProvisionerService (Fly.io lifecycle)
  - EnsureVmCommand (boot/wake VM)
  - HibernateVmCommand (idle timeout)
  - DestroyVmCommand (cleanup)
- [ ] Heartbeat scheduler (@nestjs/schedule)
  - 30-second tick loop
  - pg advisory lock (multi-replica safe)
  - Agent heartbeat interval check
  - Queue agent for execution
- [ ] InvokeHeartbeatCommand/Handler
  - Orchestrate full execution flow
  - Cost tracking
  - Budget checks
  - Live event publishing
- [ ] ApiKeyVaultService
  - AES-256 encryption
  - Store/retrieve LLM API keys
  - Validate keys against provider
- [ ] Real-time events (Redis pub/sub)
  - RedisLiveEventsService
  - Publish events to channels
  - Subscribe to channels

#### Success Criteria
- [ ] Agent can execute on Fly.io VM
- [ ] Heartbeat completes end-to-end (manual trigger)
- [ ] Cost events recorded correctly
- [ ] Budget checks prevent over-execution
- [ ] Agent callback endpoints work (task checkout, status update)
- [ ] Live events streamed to dashboard in real-time

#### Dependencies
- Phase 1 complete
- Fly.io account + API token
- Redis (Upstash)
- AWS S3 bucket
- LLM API keys (Anthropic, OpenAI, Google)

#### Risks
- Fly.io VM startup time unpredictable
- **Mitigation:** Cache warm, pre-provision VMs
- Streaming SSE events fragmented
- **Mitigation:** Robust SSE parser
- Agent state serialization breaks
- **Mitigation:** Comprehensive session codec testing

---

### Phase 3: Frontend Dashboard (Weeks 9-12)
**Focus:** React UI for company management, team building, task tracking
**Status:** Blocked on Phase 2
**Effort:** 160-200 hours

#### Deliverables
- [ ] React app setup (Vite, Tailwind, shadcn/ui)
- [ ] Authentication UI
  - Login page
  - Signup page
  - Password reset
  - Session management
- [ ] Dashboard pages
  - Company overview (metrics, recent activity)
  - Agent management (list, hire, configure, monitor)
  - Task board (create, assign, checkout, update status)
  - Org chart (hierarchical visualization)
  - Cost tracking (spend, budgets, alerts)
  - Settings (company info, preferences)
- [ ] Components
  - Agent cards (status, metrics, actions)
  - Task list (filterable, sortable)
  - Cost chart (trends over time)
  - Real-time event log
- [ ] API client (fetch wrapper)
  - Companies endpoints
  - Agents endpoints
  - Issues endpoints
  - Heartbeat endpoints
- [ ] State management (Zustand)
  - Auth state
  - Company state
  - UI state (modals, notifications)
  - Real-time event state
- [ ] WebSocket client
  - Connect to realtime gateway
  - Subscribe to company events
  - Update UI on incoming events
- [ ] Forms with validation (Zod)
  - Create company
  - Create agent
  - Create task
  - Update settings
- [ ] Error handling and loading states
- [ ] Responsive design (mobile, tablet, desktop)
- [ ] Accessibility (WCAG AA)

#### Success Criteria
- [ ] Dashboard loads within 2 seconds
- [ ] Real-time events update <1 second
- [ ] Mobile-responsive layout
- [ ] All forms validate input
- [ ] No TypeScript errors
- [ ] 90+ Lighthouse score

#### Dependencies
- Phase 2 complete
- API endpoints working

#### Risks
- WebSocket connection drops
- **Mitigation:** Reconnect with exponential backoff
- Race conditions in real-time updates
- **Mitigation:** Optimistic updates with server reconciliation
- React Query cache stale
- **Mitigation:** Proper cache invalidation strategy

---

### Phase 4: Advanced Features (Weeks 13+)
**Focus:** Templates, approvals, cost analytics, self-hiring agents
**Status:** Blocked on Phase 3
**Effort:** Varies

#### Deliverables

##### 4a. Company Templates (Week 13-14)
- [ ] Template repository (database)
- [ ] Template selection UI
- [ ] Pre-configured agent rosters
- [ ] Pre-configured goals and tasks
- [ ] Template marketplace UI
- [ ] Create custom template feature

##### 4b. Approval Workflows (Week 15-16)
- [ ] ApprovalRequest entity
- [ ] Approval board UI
- [ ] Email notifications
- [ ] Configurable approval rules
- [ ] Audit trail for approvals

##### 4c. Cost Analytics (Week 17-18)
- [ ] Cost breakdown (by agent, model, provider)
- [ ] Spending trends (daily, weekly, monthly)
- [ ] Budget forecasting
- [ ] Cost alerts and notifications
- [ ] Export reports (CSV, PDF)

##### 4d. Self-Hiring Agents (Week 19-20)
- [ ] Agent hiring proposal system
- [ ] Human approval UI
- [ ] Auto-create new agent
- [ ] On-boarding new agent (goals, permissions)
- [ ] Org structure auto-update

##### 4e. Integrations (Week 21+)
- [ ] GitHub integration (pull requests, issues)
- [ ] Slack integration (notifications, commands)
- [ ] Email notifications
- [ ] Zapier webhooks
- [ ] Custom webhook support

#### Success Criteria
- [ ] Each feature tested end-to-end
- [ ] User feedback incorporated
- [ ] Documentation complete
- [ ] Performance acceptable

---

## Timeline

```
Month 1 (Weeks 1-4):   Phase 1 - Core Platform ▓▓▓▓░░░░░░░░░░░░░░
Month 2 (Weeks 5-8):   Phase 2 - AI Execution  ░▓▓▓▓░░░░░░░░░░░░
Month 3 (Weeks 9-12):  Phase 3 - Frontend     ░░░░▓▓▓▓░░░░░░░░░░
Month 4+ (Weeks 13+):  Phase 4 - Advanced     ░░░░░░░░▓▓▓▓▓▓▓▓▓░

Initial Launch:        End of Month 3
MVP Features:          Phase 1-3 complete
Full Platform:         Phase 4 features rolling out
```

## Release Versions

### v0.1.0 (Week 4)
- Core platform, database, auth
- Manual API testing only
- **Not production**

### v0.2.0 (Week 8)
- Heartbeat scheduler working
- Single agent execution tested
- **Beta testing with trusted users**

### v0.3.0 (Week 12)
- React dashboard functional
- Multi-agent teams
- Real-time updates
- **Public beta launch** ($49/month platform fee)

### v1.0.0 (Week 16+)
- All Phase 4a-4c features
- Performance optimized
- Full documentation
- **Public GA launch**

### v1.1.0+
- Phase 4d-4e features
- Community feedback incorporated
- Scaling improvements

## Parallel Workstreams

### Deployment Infrastructure (Concurrent with all phases)
- [ ] Railway or Fly.io account setup
- [ ] PostgreSQL (Neon) provisioning
- [ ] Redis (Upstash) provisioning
- [ ] AWS S3 bucket creation
- [ ] GitHub Actions CI/CD pipeline
- [ ] Monitoring & alerting setup

### Documentation (Concurrent with all phases)
- [ ] Blueprint documentation ✅ (Complete)
- [ ] API documentation (Swagger)
- [ ] Architecture diagrams
- [ ] Developer onboarding guide
- [ ] User documentation
- [ ] FAQ & troubleshooting

### Testing (Concurrent with all phases)
- [ ] Unit test suite
- [ ] Integration test suite
- [ ] E2E test suite (Playwright)
- [ ] Performance testing
- [ ] Load testing
- [ ] Security testing

### Design & Product (Concurrent with all phases)
- [ ] Wireframes
- [ ] UI design (Figma)
- [ ] Brand guidelines
- [ ] UX research & feedback
- [ ] Accessibility audit

## Milestones

| Milestone | Target Date | Status | Criteria |
|-----------|------------|--------|----------|
| **M1: Core Platform** | Week 4 | Planned | TypeScript builds, DB schema, Auth |
| **M2: First Heartbeat** | Week 8 | Planned | Agent executes on Fly.io VM |
| **M3: Dashboard MVP** | Week 12 | Planned | React UI, real-time updates |
| **M4: Public Beta** | Week 14 | Planned | 10+ beta users, feedback collected |
| **M5: General Availability** | Week 20 | Planned | All Phase 1-3 complete, public launch |

## Success Metrics (Post-Launch)

| Metric | Target | Timeline |
|--------|--------|----------|
| **Active Companies** | 100+ | 3 months |
| **Monthly Revenue** | $5,000+ | 3 months |
| **Uptime** | 99.9% | Ongoing |
| **User Retention** | 70%+ | 6 months |
| **NPS Score** | 50+ | 6 months |
| **Agent Autonomy Rate** | 80%+ of tasks | 6 months |

## Contingencies

### Risk: Delay in Phase 1 (Database Design)
- **Impact:** All subsequent phases delayed
- **Mitigation:** Have database design review by week 1
- **Fallback:** Use simplified schema, iterate later

### Risk: Fly.io VM startup time too slow
- **Impact:** Heartbeat latency unacceptable
- **Mitigation:** Test cold-start time early (Phase 2 week 5)
- **Fallback:** Use persistent VMs (higher cost)

### Risk: Adapter implementations fail
- **Impact:** Agent execution broken
- **Mitigation:** Start with Claude only, add others later
- **Fallback:** Use process executor + manual agent commands

### Risk: Real-time WebSocket unreliable
- **Impact:** Dashboard feels sluggish
- **Mitigation:** Fallback to polling (less ideal but works)
- **Fallback:** Client-side data refresh on manual button click

### Risk: Phase 3 (Frontend) takes longer
- **Impact:** Launch delayed
- **Mitigation:** Use template components (shadcn/ui) heavily
- **Fallback:** Minimal MVP dashboard, iterate post-launch

## Product Roadmap (User-Facing)

### MVP (Week 12)
- Create AI company
- Hire agents (CEO, engineers, designers, marketers)
- Set monthly budget
- Trigger heartbeats manually
- View heartbeat logs
- Monitor agent status
- Track costs

### Q1 (Month 1-3 post-launch)
- Company templates (pre-configured teams)
- Scheduled heartbeats (automatic every N minutes)
- Approval workflows (hire, budget increase)
- Cost analytics (spend trends, forecasting)

### Q2 (Month 4-6 post-launch)
- Self-hiring agents (agents propose new hires)
- GitHub integration (agents create PRs)
- Slack integration (status notifications)
- Advanced org chart (multiple levels, departments)

### Q3+ (Month 7+ post-launch)
- Agent custom training (fine-tune on company docs)
- Skill marketplace (share/reuse agent skills)
- API for third-party integrations
- White-label version for partners

## Resources & Team

### Backend Engineers (2-3)
- CQRS architecture
- Database design
- Fly.io integration
- Cost tracking

### Frontend Engineers (1-2)
- React components
- WebSocket real-time
- User dashboard
- Forms & validation

### DevOps/Infrastructure (1)
- CI/CD pipeline
- Cloud deployment
- Monitoring & alerts
- Database backups

### Product Manager (1)
- User research
- Roadmap prioritization
- Feature decisions
- Analytics

### QA/Testing (1)
- E2E test automation
- Manual testing
- Bug reporting
- Performance testing

**Total team:** 5-8 people

## Budget Estimate

### Infrastructure (Monthly)
- API Server (Railway/Fly.io): $20
- Database (Neon Pro): $100
- Redis (Upstash Pro): $50
- S3 Storage: $10
- Monitoring (Betterstack): $0 (free tier)
- **Subtotal:** ~$180/month

### Per-Company VM (Fly.io)
- Fly.io Machine: $10-20/month per company
- Marked up 20-30% to users

### Development Costs (One-time)
- Developer time: ~1,200-1,400 hours
- At $100/hr: $120,000-140,000
- Hosting during development: $500-1,000

### Post-Launch (Monthly)
- Infrastructure: $200-500
- Ongoing development: 1-2 FTE
- Marketing: $500-2,000
- Support: $500-1,000

## Exit Criteria by Phase

### Phase 1 → Phase 2
- [ ] TypeScript builds without warnings
- [ ] All API endpoints tested and documented
- [ ] Database schema validated with sample data
- [ ] >80% unit test coverage
- [ ] Code review complete

### Phase 2 → Phase 3
- [ ] Single agent executes on Fly.io VM
- [ ] Heartbeat completes end-to-end
- [ ] Cost tracking accurate
- [ ] Real-time events working
- [ ] Agent callback endpoints tested

### Phase 3 → Phase 4
- [ ] Dashboard loads fast (<2s)
- [ ] Real-time updates working
- [ ] Mobile responsive
- [ ] Accessibility audit passed
- [ ] E2E tests passing

### Phase 4 → Public Launch
- [ ] Feature complete
- [ ] Performance optimized
- [ ] Security audit passed
- [ ] User documentation ready
- [ ] Support plan in place

---

## Questions & Decisions Needed

1. **Fly.io vs Railway for API server?**
   - Both viable, Fly.io integrates per-company VM management

2. **When to add monitoring (Datadog, Betterstack)?**
   - Phase 2, essential for tracking execution health

3. **When to add authentication for agents?**
   - Phase 2, JWT + API key injection essential

4. **Subscription or pay-as-you-go model?**
   - Decision made: Subscription ($49-99/month) + compute markup

5. **Support level at launch (MVP vs full)?**
   - MVP: Email support, community Slack
   - Phase 4: Dedicated support tier
