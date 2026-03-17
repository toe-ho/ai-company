# Phase 18: Testing

## Context Links
- [25 - Testing Strategy](../../docs/blueprint/06-infrastructure/25-testing-strategy.md)

## Overview
- **Priority:** P2
- **Status:** pending
- **Effort:** 3h
- **Description:** Vitest unit tests for critical handlers, integration tests for repository atomics, Playwright E2E for onboarding flow.
<!-- Updated: Validation Session 1 - Vitest instead of Jest -->

## Key Insights
- Unit tests: mock all deps, test handler logic only
- Integration tests: real PostgreSQL (Testcontainers or TEST_DATABASE_URL), test SQL correctness
- E2E tests: Playwright against running app, test critical user journeys
- Do NOT test: Drizzle internals, Better Auth internals, external CLIs, S3/Redis
- DO test: our guards, handlers, repository methods, API endpoints

## Architecture
```
apps/backend/
├── src/
│   ├── application/commands/issue/
│   │   └── checkout-issue-handler.spec.ts
│   ├── application/commands/heartbeat/
│   │   └── invoke-heartbeat-handler.spec.ts
│   ├── application/services/impl/
│   │   └── api-key-vault-service.spec.ts
│   ├── infrastructure/repositories/
│   │   └── issue-repository.integration.spec.ts
│   └── guard/
│       ├── board-auth-guard.spec.ts
│       └── agent-auth-guard.spec.ts
├── vitest.config.ts
└── test/
    └── setup.ts                  # Test database setup

tests/                            # E2E (Playwright)
├── playwright.config.ts
├── onboarding-wizard.e2e-spec.ts
├── dashboard.e2e-spec.ts
├── agent-creation.e2e-spec.ts
└── helpers/
    └── auth-helpers.ts           # loginAs(), createTestCompany()
```

## Related Code Files

### Files to Create
- ~8 unit test files
- ~2 integration test files
- ~3 E2E test files
- Vitest config, Playwright config, test helpers

## Implementation Steps

1. **Setup Vitest config** (`apps/backend/vitest.config.ts`)
   ```typescript
   export default {
     moduleFileExtensions: ['js', 'json', 'ts'],
     rootDir: 'src',
     testRegex: '.*\\.spec\\.ts$',
     transform: { '^.+\\.(t|j)s$': 'ts-jest' },
     collectCoverageFrom: ['**/*.(t|j)s'],
     coverageDirectory: '../coverage',
     testEnvironment: 'node',
     moduleNameMapper: {
       '^@ai-company/shared(.*)$': '<rootDir>/../../packages/shared/src$1',
     },
   };
   ```

2. **Unit test: CheckoutIssueHandler** (per blueprint 25 example)
   - Test: successful checkout -> atomicCheckout called, event published, activity logged
   - Test: 409 conflict -> atomicCheckout returns false, throws ConflictException
   - Test: wrong status -> throws UnprocessableEntityException
   - Test: issue not found -> throws NotFoundException

3. **Unit test: InvokeHeartbeatHandler**
   - Test: happy path -> validates agent, retrieves key, ensures VM, executes, records cost
   - Test: agent paused -> throws
   - Test: over budget -> throws AgentOverBudgetException
   - Test: missing API key -> throws MissingApiKeyException
   - Test: VM boot failure -> handles gracefully, sets run to 'failed'
   - Test: execution failure -> sets run to 'failed', does NOT re-throw

4. **Unit test: ApiKeyVaultService**
   - Test: encrypt/decrypt roundtrip
   - Test: stored key is ciphertext (not plaintext)
   - Test: retrieve nonexistent key -> throws

5. **Unit test: BoardAuthGuard**
   - Test: valid session -> sets request.actor, returns true
   - Test: no session -> throws UnauthorizedException
   - Test: @AllowAnonymous route -> skips auth, returns true

6. **Unit test: AgentAuthGuard**
   - Test: valid JWT -> sets actor with agentId, companyId, runId
   - Test: valid pcp_ key -> sets actor with agentId, companyId
   - Test: invalid JWT -> throws UnauthorizedException
   - Test: unknown key -> throws UnauthorizedException
   - Test: no Authorization header -> throws UnauthorizedException

7. **Integration test: IssueRepository.atomicCheckout**
   - Uses real PostgreSQL (TEST_DATABASE_URL or Testcontainers)
   - Test: concurrent checkout -> exactly one succeeds
     ```typescript
     const [r1, r2] = await Promise.all([
       repo.atomicCheckout(issueId, 'agent-1', 'run-1'),
       repo.atomicCheckout(issueId, 'agent-2', 'run-2'),
     ]);
     expect([r1, r2].filter(Boolean)).toHaveLength(1);
     ```
   - Test: release then re-checkout -> succeeds
   - Test: checkout already checked-out issue -> returns false

8. **Integration test: Budget enforcement**
   - Create agent with 100 cent budget, 95 cents spent
   - Record 10 cent cost event
   - Verify agent status changes to 'paused'

9. **Setup Playwright** (`tests/playwright.config.ts`)
   ```typescript
   export default defineConfig({
     testDir: './',
     testMatch: '**/*.e2e-spec.ts',
     use: {
       baseURL: process.env.BASE_URL ?? 'http://localhost:3100',
     },
   });
   ```

10. **E2E: Onboarding wizard** (per blueprint 25 example)
    - Navigate to /onboarding
    - Step 1: Select template
    - Step 2: Enter goal
    - Step 3: Enter API key + validate
    - Step 4: Review team (no changes)
    - Step 5: Launch
    - Verify redirect to /dashboard
    - Verify metric cards visible

11. **E2E: Dashboard rendering**
    - Login -> navigate to dashboard
    - Verify active agents panel, cost chart, activity feed visible

12. **E2E: Agent creation**
    - Login -> team page -> hire agent -> fill form -> verify card appears

13. **Create test helpers** (`tests/helpers/auth-helpers.ts`)
    - `loginAs(page, email)`: Sign in and return session
    - `createTestCompany(page)`: Run onboarding flow, return companyId

## Todo List
- [ ] Vitest config for backend
- [ ] CheckoutIssueHandler unit tests
- [ ] InvokeHeartbeatHandler unit tests
- [ ] ApiKeyVaultService unit tests
- [ ] BoardAuthGuard unit tests
- [ ] AgentAuthGuard unit tests
- [ ] IssueRepository integration tests (atomic checkout)
- [ ] Budget enforcement integration test
- [ ] Playwright config
- [ ] Onboarding wizard E2E
- [ ] Dashboard E2E
- [ ] Agent creation E2E
- [ ] Test helpers

## Success Criteria
- All unit tests pass with `turbo test`
- Atomic checkout integration test proves race condition safety
- Onboarding E2E completes full wizard flow
- CI pipeline: typecheck -> unit tests -> build -> E2E
- No mocked data used to pass tests; all tests exercise real logic

## Risk Assessment
- **Test database setup:** Need PostgreSQL available in CI. Use GitHub Actions services or Testcontainers.
- **E2E flakiness:** Use explicit waits (`waitForSelector`), not arbitrary sleeps
- **API key for E2E:** Need a test Anthropic key (or mock the validation endpoint)

## Security Considerations
- Test API keys stored as CI secrets, not in code
- Test database wiped between test runs
- E2E tests run against staging, not production

## Next Steps
- Phase 19: CI/CD runs these tests automatically
