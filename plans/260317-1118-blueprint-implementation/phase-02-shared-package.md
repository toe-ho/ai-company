# Phase 2: Shared Package

## Context Links
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md) (entity/enum lists)
- [15 - Database Design](../../docs/blueprint/04-data-and-api/15-database-design.md) (table shapes)
- [18 - API Response Schemas](../../docs/blueprint/04-data-and-api/18-api-response-schemas.md) (response types)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 1h
- **Description:** Populate `packages/shared` with entity interfaces, enums, Zod validators, API path constants, and shared types. Zero runtime deps.

## Key Insights
- Shared package consumed by ALL other packages; must remain framework-agnostic
- Entity interfaces here mirror DB table shapes but are pure TS (no Drizzle imports)
- Zod schemas used for DTO validation on both backend (ZodValidationPipe) and potentially frontend
- Use barrel exports via `src/index.ts`

## Requirements
### Functional
- All entity interfaces matching 35+ DB tables
- All enums (10+) matching domain values
- Zod validators for all DTOs
- API path constants
- IActor, IExecutionRequest, IExecutionResult, IExecutionEvent, ISessionCodec interfaces

### Non-functional
- Zero runtime dependencies (TypeScript-only, Zod is the one exception)
- All exports via barrel file

## Architecture
```
packages/shared/src/
├── index.ts                    # Barrel export
├── entities/
│   ├── index.ts
│   ├── company.ts
│   ├── agent.ts
│   ├── issue.ts
│   ├── heartbeat-run.ts
│   ├── heartbeat-run-event.ts
│   ├── goal.ts
│   ├── project.ts
│   ├── project-workspace.ts
│   ├── approval.ts
│   ├── cost-event.ts
│   ├── activity-entry.ts
│   ├── company-api-key.ts
│   ├── company-vm.ts
│   ├── agent-api-key.ts
│   ├── agent-runtime-state.ts
│   ├── agent-task-session.ts
│   ├── agent-wakeup-request.ts
│   ├── agent-config-revision.ts
│   ├── issue-comment.ts
│   ├── issue-attachment.ts
│   ├── label.ts
│   ├── approval-comment.ts
│   ├── company-template.ts
│   ├── asset.ts
│   ├── user.ts
│   ├── user-company.ts
│   └── billing-account.ts
├── enums/
│   ├── index.ts
│   ├── agent-status.ts
│   ├── issue-status.ts
│   ├── issue-priority.ts
│   ├── run-status.ts
│   ├── approval-status.ts
│   ├── actor-type.ts
│   ├── wakeup-source.ts
│   ├── adapter-type.ts
│   ├── agent-role.ts
│   ├── goal-level.ts
│   └── vm-status.ts
├── interfaces/
│   ├── index.ts
│   ├── actor.ts
│   ├── execution-request.ts
│   ├── execution-result.ts
│   ├── execution-event.ts
│   └── session-codec.ts
├── validators/
│   ├── index.ts
│   ├── company-validators.ts
│   ├── agent-validators.ts
│   ├── issue-validators.ts
│   ├── approval-validators.ts
│   ├── api-key-validators.ts
│   └── common-validators.ts
└── constants/
    ├── index.ts
    └── api-paths.ts
```

## Related Code Files

### Files to Create
All files listed in architecture above (~45 files).

### Files to Modify
- `packages/shared/src/index.ts` (barrel exports)

## Implementation Steps

1. **Create entity interfaces** (`entities/`)
   - Each file exports one interface matching the DB table columns from blueprint 15
   - Use `string` for UUIDs, `number` for cents, `Date` for timestamps
   - JSONB fields typed as `Record<string, unknown>` or specific interfaces where defined
   - Example: `company.ts` exports `ICompany { id, ownerId, name, description, status, issuePrefix, issueCounter, budgetMonthlyCents, ... }`

2. **Create enums** (`enums/`)
   - Use TypeScript `enum` or `const` objects with `as const`
   - Prefer `as const` objects for better tree-shaking:
     ```typescript
     export const AgentStatus = {
       Active: 'active',
       Paused: 'paused',
       Idle: 'idle',
       Running: 'running',
       Error: 'error',
       Terminated: 'terminated',
     } as const;
     export type AgentStatus = typeof AgentStatus[keyof typeof AgentStatus];
     ```

3. **Create interfaces** (`interfaces/`)
   - `IActor`: `{ type: ActorType, userId?, agentId?, companyId?, runId? }`
   - `IExecutionRequest`: Full payload sent to executor (from blueprint 07)
   - `IExecutionResult`: Result from executor (exitCode, usage, sessionParams, etc.)
   - `IExecutionEvent`: SSE event (status/log/system/result/error)
   - `ISessionCodec`: `{ deserialize, serialize, getDisplayId? }`

4. **Create Zod validators** (`validators/`)
   - One file per domain area
   - `company-validators.ts`: CreateCompanySchema, UpdateCompanySchema, CreateCompanyFromTemplateSchema
   - `issue-validators.ts`: CreateIssueSchema, UpdateIssueSchema, CheckoutIssueSchema, AddCommentSchema
   - `agent-validators.ts`: CreateAgentSchema, UpdateAgentSchema, WakeupAgentSchema
   - `approval-validators.ts`: CreateApprovalSchema, ResolveApprovalSchema
   - `api-key-validators.ts`: StoreApiKeySchema
   - `common-validators.ts`: PaginationSchema, UuidSchema

5. **Create API path constants** (`constants/`)
   - All endpoint paths from blueprint 17 as string constants
   - Example: `export const API_PATHS = { companies: '/companies', agents: (cid: string) => \`/companies/${cid}/agents\`, ... }`

6. **Create barrel exports**
   - Each subfolder has `index.ts` re-exporting all members
   - Root `src/index.ts` re-exports all subfolders

## Todo List
- [ ] Entity interfaces (27 files)
- [ ] Enums (11 files)
- [ ] Core interfaces (5 files)
- [ ] Zod validators (6 files)
- [ ] API path constants
- [ ] Barrel exports
- [ ] `turbo typecheck` passes

## Success Criteria
- All types importable from `@ai-company/shared`
- Zero runtime deps except `zod`
- `turbo typecheck` passes for shared + downstream packages

## Risk Assessment
- **Type drift:** Entity interfaces must stay in sync with Drizzle schemas (Phase 3). Consider generating types from Drizzle schema later.
- **Over-engineering validators:** Start with critical DTOs only, add more as controllers need them.

## Security Considerations
- No secrets or env vars in shared package
- Zod schemas enforce input validation boundaries

## Next Steps
- Phase 3: Database schemas reference these entity interfaces
- Phase 4: Domain layer imports entity types
