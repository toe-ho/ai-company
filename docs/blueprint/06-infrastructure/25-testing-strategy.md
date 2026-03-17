# 25 — Testing Strategy

Testing approach for the AI Company Platform (NestJS + Drizzle ORM + CQRS).

## Testing Stack

| Tool | Purpose |
|------|---------|
| Jest | Unit + integration test runner |
| `@nestjs/testing` | NestJS test module utilities |
| `@testcontainers/postgresql` | Spin up real PostgreSQL for integration tests |
| Playwright | E2E browser tests |
| Supertest | HTTP integration tests (optional) |

## File Naming Conventions

```
*.spec.ts          ← Unit tests (co-located with source file)
*.integration.spec.ts   ← Integration tests (repositories against real DB)
*.e2e-spec.ts      ← E2E tests (in tests/ directory)
```

Examples:
```
src/application/commands/issue/checkout-issue.handler.spec.ts
src/infrastructure/repositories/issue.repository.integration.spec.ts
tests/onboarding-wizard.e2e-spec.ts
```

## Unit Tests (Jest + NestJS Testing Module)

Unit tests run fast with all external dependencies mocked. No database, no HTTP calls.

### Testing Command Handlers

Each command handler is tested in isolation by mocking its repository and service dependencies.

```typescript
// checkout-issue.handler.spec.ts
describe('CheckoutIssueHandler', () => {
  let handler: CheckoutIssueHandler;
  let issueRepo: jest.Mocked<IIssueRepository>;
  let commandBus: jest.Mocked<CommandBus>;
  let eventBus: jest.Mocked<EventBus>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        CheckoutIssueHandler,
        { provide: 'IIssueRepository', useValue: { findByIdAndCompany: jest.fn(), atomicCheckout: jest.fn() } },
        { provide: CommandBus, useValue: { execute: jest.fn() } },
        { provide: EventBus, useValue: { publish: jest.fn() } },
      ],
    }).compile();

    handler = module.get(CheckoutIssueHandler);
    issueRepo = module.get('IIssueRepository');
    commandBus = module.get(CommandBus);
    eventBus = module.get(EventBus);
  });

  it('checks out an issue successfully', async () => {
    issueRepo.findByIdAndCompany.mockResolvedValue({ id: 'issue-1', status: 'todo', checkoutRunId: null });
    issueRepo.atomicCheckout.mockResolvedValue(true);

    const result = await handler.execute(new CheckoutIssueCommand('issue-1', 'agent-1', 'company-1', 'run-1', ['todo']));

    expect(issueRepo.atomicCheckout).toHaveBeenCalledWith('issue-1', 'agent-1', 'run-1');
    expect(eventBus.publish).toHaveBeenCalledWith(expect.any(IssueCheckedOutEvent));
    expect(result.success).toBe(true);
  });

  it('throws ConflictException when issue already checked out', async () => {
    issueRepo.findByIdAndCompany.mockResolvedValue({ id: 'issue-1', status: 'todo', checkoutRunId: null });
    issueRepo.atomicCheckout.mockResolvedValue(false); // lock failed

    await expect(
      handler.execute(new CheckoutIssueCommand('issue-1', 'agent-1', 'company-1', 'run-1', ['todo']))
    ).rejects.toThrow(ConflictException);
  });

  it('throws UnprocessableEntityException for wrong status', async () => {
    issueRepo.findByIdAndCompany.mockResolvedValue({ id: 'issue-1', status: 'done', checkoutRunId: null });

    await expect(
      handler.execute(new CheckoutIssueCommand('issue-1', 'agent-1', 'company-1', 'run-1', ['todo']))
    ).rejects.toThrow(UnprocessableEntityException);
  });
});
```

### Testing Query Handlers

```typescript
// list-agents.handler.spec.ts
describe('ListAgentsHandler', () => {
  it('returns agents for company', async () => {
    agentRepo.findByCompany.mockResolvedValue([{ id: 'a1', name: 'Alice', status: 'active' }]);
    const result = await handler.execute(new ListAgentsQuery('company-1'));
    expect(agentRepo.findByCompany).toHaveBeenCalledWith('company-1');
    expect(result).toHaveLength(1);
  });
});
```

### Testing Services

```typescript
// api-key-vault.service.spec.ts
describe('ApiKeyVaultService', () => {
  it('encrypts key on store and decrypts on retrieve', async () => {
    await service.store('company-1', 'anthropic', 'sk-ant-real-key');
    const retrieved = await service.retrieve('company-1', 'anthropic');
    expect(retrieved).toBe('sk-ant-real-key');
  });

  it('never returns plaintext key from repository', async () => {
    await service.store('company-1', 'anthropic', 'sk-ant-real-key');
    const raw = await keyRepo.findOne({ companyId: 'company-1', provider: 'anthropic' });
    expect(raw.encryptedKey).not.toBe('sk-ant-real-key'); // must be ciphertext
  });
});
```

## Integration Tests

Integration tests run against a real PostgreSQL instance (via Testcontainers or a local Docker database). They verify the actual SQL queries work correctly, including constraints and race conditions.

### Repository Integration Tests

```typescript
// issue.repository.integration.spec.ts
describe('IssueRepository (integration)', () => {
  let dataSource: DataSource;
  let repo: IssueRepository;

  beforeAll(async () => {
    // Start PostgreSQL via Testcontainers or use TEST_DATABASE_URL
    dataSource = await createTestDataSource();
    await dataSource.runMigrations();
    repo = new IssueRepository(dataSource.getRepository(IssueModel));
  });

  afterEach(async () => {
    await dataSource.query('DELETE FROM issues');
  });

  afterAll(() => dataSource.destroy());

  it('atomic checkout prevents double-checkout', async () => {
    const issue = await repo.create({ companyId: 'c1', title: 'Task', status: 'todo' });

    // Simulate two concurrent checkout attempts
    const [result1, result2] = await Promise.all([
      repo.atomicCheckout(issue.id, 'agent-1', 'run-1'),
      repo.atomicCheckout(issue.id, 'agent-2', 'run-2'),
    ]);

    // Exactly one must succeed
    expect([result1, result2].filter(Boolean)).toHaveLength(1);
    const updated = await repo.findById(issue.id);
    expect(['agent-1', 'agent-2']).toContain(updated.checkoutAgentId);
  });
});
```

### Budget Enforcement Integration Test

```typescript
it('hard-stops agent execution when over monthly budget', async () => {
  // Set agent budget to $1.00 (100 cents)
  const agent = await agentRepo.create({ companyId: 'c1', budgetMonthlyCents: 100, spentMonthlyCents: 95 });

  // Record a cost event that pushes over budget
  await commandBus.execute(new RecordCostEventCommand({ agentId: agent.id, companyId: 'c1', costCents: 10 }));

  // Agent should be auto-paused
  const updated = await agentRepo.findById(agent.id);
  expect(updated.status).toBe('paused');
});
```

## E2E Tests (Playwright)

E2E tests run against a fully deployed staging environment. Located in `tests/`.

### Onboarding Wizard Flow

```typescript
// tests/onboarding-wizard.e2e-spec.ts
test('complete onboarding wizard creates a company', async ({ page }) => {
  await page.goto('/onboarding');

  // Step 1: Select template
  await page.click('[data-testid="template-saas-startup"]');
  await page.click('[data-testid="next-step"]');

  // Step 2: Enter goal
  await page.fill('[data-testid="goal-input"]', 'Build a task management app');
  await page.click('[data-testid="next-step"]');

  // Step 3: API key setup
  await page.fill('[data-testid="api-key-input"]', process.env.TEST_ANTHROPIC_KEY);
  await page.click('[data-testid="validate-key"]');
  await page.waitForSelector('[data-testid="key-valid-badge"]');
  await page.click('[data-testid="next-step"]');

  // Step 4: Review team (no changes)
  await page.click('[data-testid="next-step"]');

  // Step 5: Launch
  await page.click('[data-testid="launch-button"]');
  await page.waitForURL('*/dashboard');

  // Verify dashboard loaded with new company
  await expect(page.locator('[data-testid="metric-card-agents"]')).toBeVisible();
});
```

### Dashboard Rendering

```typescript
// tests/dashboard.e2e-spec.ts
test('dashboard shows active agents and recent activity', async ({ page }) => {
  await loginAs(page, 'test@example.com');
  await page.goto('/test-company/dashboard');

  await expect(page.locator('[data-testid="active-agents-panel"]')).toBeVisible();
  await expect(page.locator('[data-testid="cost-chart"]')).toBeVisible();
  await expect(page.locator('[data-testid="activity-feed"]')).toBeVisible();
});
```

### Agent Creation from Template

```typescript
// tests/agent-creation.e2e-spec.ts
test('hire agent from template creates agent card', async ({ page }) => {
  await loginAs(page, 'test@example.com');
  await page.goto('/test-company/team');
  await page.click('[data-testid="hire-agent-button"]');
  await page.click('[data-testid="template-engineer"]');
  await page.fill('[data-testid="agent-name"]', 'Bob the Engineer');
  await page.click('[data-testid="hire-confirm"]');

  await expect(page.locator('text=Bob the Engineer')).toBeVisible();
});
```

## What NOT to Test

- **Adapter CLIs** (claude, codex, etc.) — these are external binaries, not our code. We test that the adapter layer calls them with the right arguments, not their internal behavior.
- **Drizzle ORM internals** — trust the library.
- **Fly.io Machines API** — mock `FlyioClient` in unit tests; test the provisioner logic only.
- **Better Auth internals** — trust the library; test our guards and decorators instead.
- **S3/Redis** — mock `IStorageService` and `ILiveEventsService` in unit tests.

## CI Pipeline

Tests run in this order, gated sequentially:

```
typecheck → unit tests → build → E2E tests
```

```yaml
# .github/workflows/ci.yml
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - run: turbo typecheck

  unit-tests:
    needs: typecheck
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: test, POSTGRES_PASSWORD: test }
    steps:
      - run: turbo test

  build:
    needs: unit-tests
    runs-on: ubuntu-latest
    steps:
      - run: turbo build

  e2e-tests:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npx playwright install --with-deps
      - run: turbo test:e2e
        env: { BASE_URL: ${{ vars.STAGING_URL }}, TEST_ANTHROPIC_KEY: ${{ secrets.TEST_ANTHROPIC_KEY }} }
```

**Rule:** Never merge if any step fails. Never use `--passWithNoTests`. Never mock data to make tests pass — fix the underlying issue.
