# Phase 12: Presentation Layer - Controllers & DTOs

## Context Links
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md) (presentation/ section)
- [17 - API Design](../../docs/blueprint/04-data-and-api/17-api-design.md)
- [18 - API Response Schemas](../../docs/blueprint/04-data-and-api/18-api-response-schemas.md)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 4h
- **Description:** All HTTP controllers (18) and DTOs (25+). Controllers dispatch to commandBus/queryBus only -- zero business logic.

## Key Insights
- Controllers grouped by auth surface: board (session), agent (JWT/API key), public (none), internal (none)
- Controllers are thin: extract params -> dispatch command/query -> return result
- DTOs validated via `@UsePipes(new ZodValidationPipe(schema))`
- Board controllers use @UseGuards(CompanyAccessGuard) + optionally @Roles
- Agent controllers use @UseGuards(AgentAuthGuard, CompanyAccessGuard)
- Public controllers use @AllowAnonymous

## Architecture
```
apps/backend/src/presentation/
├── controllers/
│   ├── dto/
│   │   ├── company/
│   │   │   ├── create-company-dto.ts
│   │   │   ├── create-company-from-template-dto.ts
│   │   │   └── update-company-dto.ts
│   │   ├── agent/
│   │   │   ├── create-agent-dto.ts
│   │   │   ├── update-agent-dto.ts
│   │   │   └── wakeup-agent-dto.ts
│   │   ├── issue/
│   │   │   ├── create-issue-dto.ts
│   │   │   ├── update-issue-dto.ts
│   │   │   ├── checkout-issue-dto.ts
│   │   │   └── add-comment-dto.ts
│   │   ├── approval/
│   │   │   ├── create-approval-dto.ts
│   │   │   └── resolve-approval-dto.ts
│   │   ├── api-key-vault/
│   │   │   └── store-api-key-dto.ts
│   │   ├── goal/
│   │   │   ├── create-goal-dto.ts
│   │   │   └── update-goal-dto.ts
│   │   ├── project/
│   │   │   ├── create-project-dto.ts
│   │   │   └── create-workspace-dto.ts
│   │   └── common/
│   │       └── pagination-dto.ts
│   └── impl/
│       ├── board/
│       │   ├── board-company-controller.ts
│       │   ├── board-agent-controller.ts
│       │   ├── board-issue-controller.ts
│       │   ├── board-goal-controller.ts
│       │   ├── board-project-controller.ts
│       │   ├── board-approval-controller.ts
│       │   ├── board-cost-controller.ts
│       │   ├── board-activity-controller.ts
│       │   ├── board-dashboard-controller.ts
│       │   ├── board-api-key-vault-controller.ts
│       │   ├── board-vm-controller.ts
│       │   └── board-template-controller.ts
│       ├── agent/
│       │   ├── agent-self-controller.ts
│       │   ├── agent-issue-controller.ts
│       │   └── agent-approval-controller.ts
│       ├── internal/
│       │   └── health-check-controller.ts
│       └── public/
│           ├── auth-controller.ts
│           └── public-template-controller.ts
└── gateways/
    └── (Phase 13)
```

## Related Code Files

### Files to Create
~18 controllers + ~20 DTO files = ~38 files.

## Implementation Steps

1. **Create DTO files**
   Each DTO exports a Zod schema + inferred TypeScript type:
   ```typescript
   // dto/company/create-company-dto.ts
   import { z } from 'zod';
   export const CreateCompanySchema = z.object({
     name: z.string().min(1).max(100),
     description: z.string().max(500).optional(),
   });
   export type CreateCompanyDto = z.infer<typeof CreateCompanySchema>;
   ```
   Create DTOs for all endpoints per blueprint 17/18.

2. **Board controllers** (12 controllers, session auth)

   **BoardCompanyController** (`/api/companies`)
   - `GET /` -> ListCompaniesQuery(actor.userId)
   - `POST /` -> CreateCompanyCommand(actor.userId, dto)
   - `POST /from-template` -> CreateCompanyFromTemplateCommand(actor.userId, dto)
   - `GET /:id` -> GetCompanyQuery(id)
   - `PATCH /:id` -> UpdateCompanyCommand(id, dto)
   - `DELETE /:id` -> DeleteCompanyCommand(id) [@Roles('owner')]

   **BoardAgentController** (`/api/companies/:companyId/agents` + `/api/agents/:id`)
   - `GET /companies/:cid/agents` -> ListAgentsQuery(cid)
   - `POST /companies/:cid/agents` -> CreateAgentCommand(cid, dto)
   - `GET /agents/:id` -> GetAgentQuery(id)
   - `PATCH /agents/:id` -> UpdateAgentCommand(id, dto)
   - `POST /agents/:id/pause` -> PauseAgentCommand(id)
   - `POST /agents/:id/resume` -> ResumeAgentCommand(id)
   - `POST /agents/:id/terminate` -> TerminateAgentCommand(id)
   - `POST /agents/:id/wakeup` -> WakeupAgentCommand(id, dto)
   - `POST /agents/:id/heartbeat/invoke` -> InvokeHeartbeatCommand(id)
   - `GET /agents/:id/runtime-state` -> GetRuntimeStateQuery(id)
   - `GET /agents/:id/config-revisions` -> ListConfigRevisionsQuery(id)
   - `POST /agents/:id/config-revisions/:rid/rollback` -> RollbackAgentConfigCommand(id, rid)
   - `GET /companies/:cid/org` -> GetOrgTreeQuery(cid)

   **BoardIssueController** (`/api/companies/:cid/issues` + `/api/issues/:id`)
   - `GET /companies/:cid/issues` -> ListIssuesQuery(cid, filters)
   - `POST /companies/:cid/issues` -> CreateIssueCommand(cid, dto)
   - `GET /issues/:id` -> GetIssueQuery(id)
   - `PATCH /issues/:id` -> UpdateIssueCommand(id, dto)
   - `GET /issues/:id/comments` -> ListCommentsQuery(id, pagination)
   - `POST /issues/:id/comments` -> AddCommentCommand(id, dto)
   - `POST /issues/:id/attachments` -> UploadAttachmentCommand(id, file)
   - `GET /issues/:id/heartbeat-context` -> GetHeartbeatContextQuery(id)

   **BoardGoalController, BoardProjectController, BoardApprovalController**: Standard CRUD patterns.

   **BoardCostController** (`/api/companies/:cid/costs`)
   - `GET /` -> GetCostSummaryQuery(cid, dateRange)

   **BoardActivityController** (`/api/companies/:cid/activity`)
   - `GET /` -> ListActivityQuery(cid, pagination)

   **BoardDashboardController** (`/api/companies/:cid/dashboard`)
   - `GET /summary` -> GetDashboardSummaryQuery(cid)

   **BoardApiKeyVaultController** (`/api/companies/:cid/api-keys`)
   - `GET /` -> list stored keys (masked)
   - `POST /` -> StoreApiKeyCommand(cid, dto) [@Roles('owner', 'admin')]
   - `DELETE /:kid` -> RevokeApiKeyCommand(cid, kid) [@Roles('owner', 'admin')]
   - `POST /:kid/validate` -> ValidateApiKeyCommand(cid, kid)

   **BoardVmController** (`/api/companies/:cid/vm`)
   - `GET /` -> GetVmStatusQuery(cid)
   - `POST /wake` -> EnsureVmCommand(cid)
   - `POST /hibernate` -> HibernateVmCommand(cid)

   **BoardTemplateController** (`/api/templates`) -- authenticated version
   - `GET /` -> ListTemplatesQuery()
   - `GET /:slug` -> GetTemplateQuery(slug)

3. **Agent controllers** (3 controllers, JWT/API key auth)

   **AgentSelfController** (`/api/agents/me`)
   - `@UseGuards(AgentAuthGuard, CompanyAccessGuard)`
   - `GET /` -> GetAgentMeQuery(actor.agentId)
   - `GET /inbox-lite` -> GetAgentInboxQuery(actor.agentId)

   **AgentIssueController** (`/api/issues/:id`)
   - `@UseGuards(AgentAuthGuard, CompanyAccessGuard)`
   - `POST /:id/checkout` -> CheckoutIssueCommand(id, actor.agentId, actor.companyId, runId, dto)
   - `POST /:id/release` -> ReleaseIssueCommand(id)
   - `PATCH /:id` -> UpdateIssueCommand(id, dto, actor, runId)
   - `POST /:id/comments` -> AddCommentCommand(id, dto, actor, runId)
   - `POST /` -> CreateIssueCommand(actor.companyId, dto) [subtask creation]

   **AgentApprovalController** (`/api/companies/:cid/approvals`)
   - `@UseGuards(AgentAuthGuard, CompanyAccessGuard)`
   - `POST /` -> CreateApprovalCommand(cid, dto, actor.agentId)

4. **Public controllers** (no auth)

   **AuthController** (`/api/auth`)
   - `@AllowAnonymous()` on all routes
   - `POST /sign-up/email` -> delegate to AuthService
   - `POST /sign-in/email` -> delegate to AuthService
   - `GET /get-session` -> delegate to AuthService
   - `ALL /api/auth/*` -> catch-all proxy to Better Auth

   **PublicTemplateController** (`/api/templates`)
   - `@AllowAnonymous()`
   - `GET /` -> ListTemplatesQuery(publicOnly: true)
   - `GET /:slug` -> GetTemplateQuery(slug)

5. **Internal controllers**

   **HealthCheckController** (`/api/health`)
   - `@AllowAnonymous()`
   - `GET /` -> `{ status: 'ok', version, authReady, features }`

6. **Register controllers in `ApiModule`**
   ```typescript
   @Module({
     controllers: [
       ...BoardControllers,
       ...AgentControllers,
       ...PublicControllers,
       HealthCheckController,
     ],
   })
   export class ApiModule {}
   ```

## Todo List
- [ ] DTO files (~20)
- [ ] Board controllers (12)
- [ ] Agent controllers (3)
- [ ] Public controllers (2)
- [ ] Internal controller (1)
- [ ] Register all in ApiModule
- [ ] typecheck passes
- [ ] Verify endpoint paths match blueprint 17

## Success Criteria
- All endpoints from blueprint 17 have corresponding controller methods
- DTOs validate with Zod schemas
- Controllers contain zero business logic (dispatch only)
- Auth guards correctly applied per controller group
- `turbo typecheck` passes

## Risk Assessment
- **Route conflicts:** board and agent controllers both handle `/api/issues/:id` -- use different controller classes with same route prefix but different guards
- **File upload:** UploadAttachment needs `@UseInterceptors(FileInterceptor)` from @nestjs/platform-express
- **Better Auth proxy:** Auth controller must forward raw request to Better Auth handler

## Security Considerations
- Board routes: session auth (global BoardAuthGuard)
- Agent routes: explicit @UseGuards(AgentAuthGuard)
- Public routes: @AllowAnonymous
- Company scope: CompanyAccessGuard on all company-scoped routes
- Role-based: @Roles on destructive operations (delete company, manage API keys)

## Next Steps
- Phase 13: RealtimeGateway + Scheduler registered alongside controllers
- Phase 16: Frontend consumes these endpoints
