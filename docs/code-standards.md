# Code Standards & Best Practices

This document establishes coding conventions and architectural patterns for AI Company Platform v2.

---

## TypeScript & General Standards

### Strict Type Safety
```typescript
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitThis": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

### Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Files (TS/JS) | kebab-case | `execution-engine.service.ts` |
| Classes/Types | PascalCase | `ExecutionEngine`, `IRepository` |
| Interfaces | PascalCase (no `I` prefix) | `Repository` (not `IRepository`) |
| Variables/Functions | camelCase | `getAgentById`, `executionId` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Enums | PascalCase (item: UPPER_SNAKE_CASE) | `enum Role { CEO, CTO, ENGINEER }` |
| DB Columns | snake_case | `company_id`, `created_at` |
| Package Names | kebab-case with scope | `@ai-company/shared`, `@ai-company/adapters` |

### File Organization

**Max 200 lines per file.** Exceed this → split into modules.

**Single Responsibility:** Each file exports one primary entity (class, function set, or enum).

```typescript
// ✅ Good: Single responsibility
// execution-engine.service.ts
export class ExecutionEngineService { }

// ❌ Bad: Multiple responsibilities in one file
// services.ts
export class ExecutionEngineService { }
export class CostCalculatorService { }
export class TaskQueueService { }
```

### Imports & Organization

**Order:**
1. Node.js builtins (`fs`, `path`, `http`)
2. Third-party packages (`nestjs`, `zod`, `postgres`)
3. Absolute imports (workspace packages, `@ai-company/*`)
4. Relative imports (`../`, `./`)
5. Type imports (separate with `type` keyword)

```typescript
import * as fs from 'fs';
import { Injectable } from '@nestjs/common';
import { z } from 'zod';
import { CompanyRepository } from '@ai-company/shared/repositories';
import { executeTask } from '../services/executor';
import type { ExecutionRequest } from '../types';
```

### Error Handling

**Always use try-catch or Promise.catch(). No silent failures.**

```typescript
// ✅ Good: Explicit error handling
async execute(request: ExecutionRequest): Promise<ExecutionResult> {
  try {
    const result = await adapter.run(request);
    return result;
  } catch (err) {
    if (err instanceof TimeoutError) {
      logger.warn('Execution timeout', { requestId: request.id });
      throw new ExecutionTimeoutException();
    }
    logger.error('Execution failed', { error: err, requestId: request.id });
    throw new ExecutionFailedException();
  }
}

// ❌ Bad: Silent failure
async execute(request: ExecutionRequest): Promise<ExecutionResult> {
  try {
    return await adapter.run(request);
  } catch {
    return null; // Silent fail
  }
}
```

### Security Standards

**API Keys & Secrets:**
- Never log credentials
- Always encrypt at rest (in DB)
- Use environment variables, never hardcoded
- Rotate API keys on schedule

```typescript
// ✅ Good: Encrypted secrets
async getApiKey(companyId: string): Promise<string> {
  const encrypted = await this.repo.getEncryptedKey(companyId);
  return decrypt(encrypted, this.encryptionKey);
}

// ❌ Bad: Logging credentials
logger.debug('API key:', apiKey);
```

**Multi-tenant Isolation:**
```typescript
// ✅ Always filter by companyId
async getAgents(companyId: string): Promise<Agent[]> {
  return this.db.query('SELECT * FROM agents WHERE company_id = ?', [companyId]);
}

// ❌ Bad: Missing tenant filter
async getAgents(): Promise<Agent[]> {
  return this.db.query('SELECT * FROM agents'); // Leaks all agents
}
```

---

## Backend (NestJS + Clean Architecture)

### Architecture Layers

**4 layers, no cross-layer violations:**

```
Presentation
    ↓
Application
    ↓
Infrastructure  ↔  Domain
```

**Rules:**
- **Domain** layer: Zero framework imports. Pure TypeScript. Entities, interfaces, enums only.
- **Infrastructure** layer: Drizzle schemas, repositories, external clients. Can import from domain.
- **Application** layer: Commands, queries, services. Can import from domain + infrastructure.
- **Presentation** layer: Controllers, DTOs, guards. Can import from application.

### Domain Layer

**Entity Definition:**
```typescript
// src/domain/entities/company.entity.ts
export class Company {
  id: string;
  name: string;
  ownerId: string;
  goal: string;
  budgetLimit: number;
  status: CompanyStatus;
  createdAt: Date;

  constructor(data: CompanyData) {
    Object.assign(this, data);
  }

  // Domain logic only
  canHireAgent(): boolean {
    return this.status === CompanyStatus.ACTIVE;
  }

  getRemainingBudget(): number {
    return this.budgetLimit - this.spentAmount;
  }
}
```

**Repository Interface (Domain):**
```typescript
// src/domain/repositories/company.repository.interface.ts
export interface ICompanyRepository {
  save(company: Company): Promise<void>;
  findById(id: string, companyId: string): Promise<Company | null>;
  update(id: string, companyId: string, data: Partial<Company>): Promise<void>;
}
```

**Enums & Types:**
```typescript
// src/domain/enums/agent-role.enum.ts
export enum AgentRole {
  CEO = 'CEO',
  CTO = 'CTO',
  ENGINEER = 'ENGINEER',
  DESIGNER = 'DESIGNER',
  MARKETER = 'MARKETER',
}

// src/domain/types.ts
export type ExecutionResult = {
  success: boolean;
  output: string;
  tokensUsed: number;
  costUSD: number;
};
```

**Exceptions (Domain):**
```typescript
// src/domain/exceptions/company-not-found.exception.ts
export class CompanyNotFoundException extends Error {
  constructor(companyId: string) {
    super(`Company ${companyId} not found`);
  }
}
```

### Infrastructure Layer

**Drizzle Schema:**
```typescript
// src/infrastructure/database/schema.ts
import { pgTable, text, uuid, integer, boolean, timestamp, jsonb } from 'drizzle-orm/pg-core';

export const companies = pgTable('companies', {
  id: uuid('id').primaryKey().defaultRandom(),
  ownerId: uuid('owner_id').notNull(),
  name: text('name').notNull(),
  goal: text('goal').notNull(),
  budgetLimit: integer('budget_limit').notNull(),
  status: text('status').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const agents = pgTable('agents', {
  id: uuid('id').primaryKey().defaultRandom(),
  companyId: uuid('company_id').notNull().references(() => companies.id),
  name: text('name').notNull(),
  role: text('role').notNull(),
  adapterType: text('adapter_type').notNull(),
  budgetLimit: integer('budget_limit').notNull(),
  config: jsonb('config'), // Flexible JSON for adapter configs
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

**Repository Implementation:**
```typescript
// src/infrastructure/repositories/company.repository.ts
@Injectable()
export class CompanyRepository implements ICompanyRepository {
  constructor(private db: Database) {}

  async save(company: Company): Promise<void> {
    const { id, name, ownerId, goal, budgetLimit, status, createdAt } = company;
    await this.db.insert(companies).values({
      id, name, ownerId, goal, budgetLimit, status, createdAt,
    });
  }

  async findById(id: string, companyId: string): Promise<Company | null> {
    const row = await this.db.query.companies.findFirst({
      where: (c) => and(eq(c.id, id), eq(c.companyId, companyId)), // Always filter by tenant
    });
    return row ? new Company(row) : null;
  }
}
```

### Application Layer (CQRS)

**Command (Mutation):**
```typescript
// src/application/commands/create-company.command.ts
export class CreateCompanyCommand {
  constructor(
    readonly ownerId: string,
    readonly name: string,
    readonly goal: string,
    readonly budgetLimit: number,
  ) {}
}

// src/application/commands/create-company.handler.ts
@CommandHandler(CreateCompanyCommand)
@Injectable()
export class CreateCompanyHandler {
  constructor(
    private repo: ICompanyRepository,
    private eventBus: EventBus,
  ) {}

  async execute(command: CreateCompanyCommand): Promise<Company> {
    const company = new Company({
      id: generateId(),
      ownerId: command.ownerId,
      name: command.name,
      goal: command.goal,
      budgetLimit: command.budgetLimit,
      status: CompanyStatus.ACTIVE,
      createdAt: new Date(),
    });

    await this.repo.save(company);
    this.eventBus.publish(new CompanyCreatedEvent(company.id, company.name));

    return company;
  }
}
```

**Query (Read):**
```typescript
// src/application/queries/get-agents.query.ts
export class GetAgentsQuery {
  constructor(readonly companyId: string) {}
}

// src/application/queries/get-agents.handler.ts
@QueryHandler(GetAgentsQuery)
@Injectable()
export class GetAgentsHandler {
  constructor(private repo: IAgentRepository) {}

  async execute(query: GetAgentsQuery): Promise<Agent[]> {
    return this.repo.findByCompanyId(query.companyId);
  }
}
```

**Service (Business Logic):**
```typescript
// src/application/services/execution-engine.service.ts
@Injectable()
export class ExecutionEngineService {
  constructor(
    private adapterRegistry: AdapterRegistry,
    private costCalculator: CostCalculatorService,
    private flyApi: FlyApiClient,
  ) {}

  async dispatch(request: ExecutionRequest): Promise<ExecutionResult> {
    const adapter = this.adapterRegistry.get(request.adapterType);
    const vmUrl = await this.getCompanyVmUrl(request.companyId);

    const response = await adapter.execute({
      ...request,
      vmUrl,
    });

    const cost = this.costCalculator.calculate(response);
    return { ...response, costUSD: cost };
  }
}
```

### Presentation Layer (Controllers & DTOs)

**DTO (Data Transfer Object):**
```typescript
// src/presentation/dtos/create-company.dto.ts
export class CreateCompanyDTO {
  @IsString()
  name: string;

  @IsString()
  goal: string;

  @IsNumber()
  @Min(100)
  @Max(100000)
  budgetLimit: number;
}
```

**Controller:**
```typescript
// src/presentation/controllers/company.controller.ts
@Controller('companies')
@UseGuards(JwtAuthGuard)
export class CompanyController {
  constructor(private commandBus: CommandBus, private queryBus: QueryBus) {}

  @Post()
  async createCompany(
    @Body() dto: CreateCompanyDTO,
    @CurrentUser() user: AuthUser,
  ): Promise<CompanyResponseDTO> {
    const command = new CreateCompanyCommand(
      user.id,
      dto.name,
      dto.goal,
      dto.budgetLimit,
    );
    const company = await this.commandBus.execute(command);
    return toCompanyResponseDTO(company);
  }

  @Get(':id')
  async getCompany(
    @Param('id') id: string,
    @CurrentUser() user: AuthUser,
  ): Promise<CompanyResponseDTO> {
    const company = await this.queryBus.execute(
      new GetCompanyQuery(id, user.companyId),
    );
    if (!company) throw new NotFoundException();
    return toCompanyResponseDTO(company);
  }
}
```

**Guard (Authentication):**
```typescript
// src/presentation/guards/jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>();
    const token = request.headers.authorization?.split(' ')[1];

    if (!token) throw new UnauthorizedException('Missing token');

    try {
      const payload = this.jwtService.verify(token);
      request.user = payload; // Attach to request
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```

### Zod Validation

```typescript
// src/infrastructure/validators/company.schema.ts
export const createCompanySchema = z.object({
  name: z.string().min(1).max(100),
  goal: z.string().min(10).max(500),
  budgetLimit: z.number().min(100).max(100000),
});

export type CreateCompanyInput = z.infer<typeof createCompanySchema>;

// In service
async createCompany(data: unknown): Promise<Company> {
  const validated = createCompanySchema.parse(data); // Throws if invalid
  // ...
}
```

---

## Frontend (React)

### Component Structure

**Naming:**
```typescript
// Pages use "Page" suffix
export function DashboardPage() { }  // src/pages/dashboard/DashboardPage.tsx

// Components use PascalCase
export function TaskCard() { }        // src/components/TaskCard.tsx

// Hooks use "use" prefix
export function useCompany() { }      // src/hooks/useCompany.ts

// Utils are lowercase
export function formatDate() { }      // src/utils/format-date.util.ts
```

**Component Template:**
```typescript
// src/components/TaskCard.tsx
import { FC } from 'react';
import type { Task } from '@ai-company/shared';

interface TaskCardProps {
  task: Task;
  onAssign?: (agentId: string) => void;
}

export const TaskCard: FC<TaskCardProps> = ({ task, onAssign }) => {
  return (
    <div className="border rounded-lg p-4">
      <h3 className="font-bold">{task.title}</h3>
      <p className="text-sm text-gray-600">{task.description}</p>
      <button onClick={() => onAssign?.(task.assignedTo)}>Assign</button>
    </div>
  );
};
```

### React Hooks

**Custom Hooks Pattern:**
```typescript
// src/hooks/useCompany.ts
export function useCompany(companyId: string) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['company', companyId],
    queryFn: () => api.getCompany(companyId),
  });

  return { company: data, isLoading, error };
}
```

### API Client

```typescript
// src/services/api.service.ts
const apiClient = axios.create({ baseURL: import.meta.env.VITE_API_URL });

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('authToken');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export const api = {
  getCompany: (id: string) => apiClient.get(`/companies/${id}`),
  createAgent: (companyId: string, data: unknown) =>
    apiClient.post(`/companies/${companyId}/agents`, data),
};
```

### React Query Integration

```typescript
// src/pages/dashboard/DashboardPage.tsx
export function DashboardPage() {
  const { company, isLoading, error } = useCompany(useParams().companyId!);
  const { data: agents } = useQuery({
    queryKey: ['agents', company?.id],
    queryFn: () => api.getAgents(company!.id),
    enabled: !!company,
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorBoundary error={error} />;

  return (
    <div>
      <h1>{company.name}</h1>
      {agents?.map((a) => <AgentCard key={a.id} agent={a} />)}
    </div>
  );
}
```

### Form Handling

```typescript
// src/pages/agent-config/AgentConfigPage.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { updateAgentSchema } from '@ai-company/shared';

export function AgentConfigPage({ agentId }: { agentId: string }) {
  const form = useForm({
    resolver: zodResolver(updateAgentSchema),
    defaultValues: { role: '', budgetLimit: 1000 },
  });

  const mutation = useMutation({
    mutationFn: (data) => api.updateAgent(agentId, data),
    onSuccess: () => {
      toast.success('Agent updated');
      queryClient.invalidateQueries({ queryKey: ['agent', agentId] });
    },
  });

  return (
    <form onSubmit={form.handleSubmit((data) => mutation.mutate(data))}>
      <input {...form.register('role')} placeholder="Role" />
      <button type="submit" disabled={mutation.isPending}>
        Save
      </button>
    </form>
  );
}
```

### Styling (Tailwind CSS 4 + shadcn/ui)

```typescript
// ✅ Use utility classes for layout
<div className="flex flex-col gap-4 p-4 border rounded-lg">
  <h2 className="text-lg font-bold">Tasks</h2>
  <button className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700">
    Add Task
  </button>
</div>

// ✅ Use shadcn/ui components for complex widgets
<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Create Agent</DialogTitle>
    </DialogHeader>
    {/* Form inside */}
  </DialogContent>
</Dialog>

// ❌ Avoid inline styles
<div style={{ display: 'flex', gap: '16px', padding: '16px' }}>
```

---

## Database (Drizzle ORM)

### Query Patterns

**Safe, type-safe queries:**
```typescript
// ✅ Good: Type-safe, parameterized
const agents = await db.query.agents.findMany({
  where: eq(agents.companyId, companyId),
});

const agent = await db.query.agents.findFirst({
  where: and(
    eq(agents.id, agentId),
    eq(agents.companyId, companyId), // Always filter by tenant
  ),
});

// ❌ Bad: Raw SQL, SQL injection risk
const agents = await db.execute(
  sql`SELECT * FROM agents WHERE company_id = ${companyId}`
);
```

### Migrations (drizzle-kit auto-generate)

Migrations are **auto-generated** by `drizzle-kit` from schema diffs. Never write migration SQL manually.

**Workflow:**
```bash
# 1. Edit schema files in src/infrastructure/persistence/schema/*.ts
# 2. Generate migration (diffs current schema vs last snapshot)
pnpm drizzle-kit generate

# 3. Review generated SQL in drizzle/ directory
# 4. Apply migration to database
pnpm drizzle-kit migrate

# Dev only: push schema directly (no migration file, for rapid iteration)
pnpm drizzle-kit push
```

**Config (`drizzle.config.ts` at backend root):**
```typescript
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/infrastructure/persistence/schema/*.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

**Generated migration structure:**
```
apps/backend/drizzle/
├── 0000_init.sql
├── 0001_add_agents.sql
├── meta/
│   ├── 0000_snapshot.json    ← schema snapshot for diffing
│   └── _journal.json         ← migration journal
└── ...
```

**Rules:**
- Always run `drizzle-kit generate` after schema changes
- Review generated SQL before applying — check for destructive ops (DROP COLUMN, etc.)
- Commit migration files alongside schema changes in the same PR
- Use `drizzle-kit push` only in local dev, never in staging/production
- For production: apply via `drizzle-kit migrate` in CI/CD pipeline
```

---

## Testing

### Unit Tests (Vitest)

**File naming:**
```
src/
├── services/
│   ├── execution-engine.service.ts
│   └── execution-engine.service.test.ts
```

**Test template:**
```typescript
// src/application/services/execution-engine.service.test.ts
import { describe, it, expect, vi } from 'vitest';
import { ExecutionEngineService } from './execution-engine.service';
import type { ExecutionRequest, ExecutionResult } from '../types';

describe('ExecutionEngineService', () => {
  let service: ExecutionEngineService;

  beforeEach(() => {
    service = new ExecutionEngineService(
      mockAdapterRegistry,
      mockCostCalculator,
      mockFlyApi,
    );
  });

  it('should execute task and calculate cost', async () => {
    const request: ExecutionRequest = {
      companyId: 'comp-1',
      agentId: 'agent-1',
      taskId: 'task-1',
      adapterType: 'CLAUDE_CLI',
      prompt: 'Hello world',
    };

    const result = await service.dispatch(request);

    expect(result.success).toBe(true);
    expect(result.costUSD).toBeGreaterThan(0);
  });

  it('should retry on timeout', async () => {
    const request = { /* ... */ };
    vi.spyOn(adapter, 'execute').mockRejectedValueOnce(new TimeoutError());

    const result = await service.dispatch(request);

    expect(adapter.execute).toHaveBeenCalledTimes(2); // Retry once
  });
});
```

### E2E Tests (Playwright)

**File naming:**
```
tests/
├── auth.spec.ts
├── company-creation.spec.ts
├── task-execution.spec.ts
└── ...
```

**Test template:**
```typescript
// tests/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('should sign up and login', async ({ page }) => {
    await page.goto('http://localhost:5173');
    await page.click('text=Sign Up');

    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'SecurePass123!');
    await page.click('button:has-text("Create Account")');

    await expect(page).toHaveURL(/\/onboarding/);
  });
});
```

---

## Error Handling & Logging

### Custom Exceptions

```typescript
// src/domain/exceptions/execution.exception.ts
export class ExecutionException extends Error {
  constructor(
    message: string,
    readonly code: string,
    readonly statusCode: number = 500,
  ) {
    super(message);
  }
}

export class ExecutionTimeoutException extends ExecutionException {
  constructor() {
    super('Execution timeout', 'EXECUTION_TIMEOUT', 408);
  }
}
```

### Global Exception Filter

```typescript
// src/presentation/filters/global-exception.filter.ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private logger: Logger) {}

  catch(exception: unknown, host: ExecutionContext) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let statusCode = 500;
    let message = 'Internal server error';

    if (exception instanceof ExecutionException) {
      statusCode = exception.statusCode;
      message = exception.message;
    }

    this.logger.error(`[${request.method}] ${request.url}`, exception);

    response.status(statusCode).json({
      statusCode,
      message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

### Logging

```typescript
// Use structured logging
this.logger.log('Company created', {
  companyId: company.id,
  ownerId: company.ownerId,
  timestamp: new Date(),
});

this.logger.warn('Agent budget exceeded', {
  agentId: agent.id,
  spent: agent.spent,
  limit: agent.budgetLimit,
});

this.logger.error('Execution failed', {
  error: err.message,
  taskId: task.id,
  stack: err.stack,
});
```

---

## Git & Commit Standards

### Commit Messages

**Format:** Conventional Commits (`type(scope): message`)

```
feat(adapter): add Claude CLI adapter implementation
fix(executor): retry on network timeout
docs(readme): update deployment instructions
test(auth): add JWT validation tests
refactor(repositories): extract query builders to utils
chore(deps): upgrade drizzle-orm to 0.30.0
```

### Branch Naming

```
feature/execute-task-implementation
bugfix/agent-budget-overflow
docs/api-documentation
refactor/clean-architecture-layers
test/e2e-integration-tests
```

---

## Code Review Checklist

Before pushing, ensure:
- [ ] TypeScript compiles without errors (`tsc --noEmit`)
- [ ] All tests pass (`pnpm test`)
- [ ] No `any` types (except unavoidable third-party)
- [ ] No credentials in code (use .env)
- [ ] Multi-tenant filters on all DB queries
- [ ] Error handling for all async operations
- [ ] No console.log (use logger)
- [ ] Meaningful commit message
- [ ] File size < 200 LOC
- [ ] Comments for complex logic only
- [ ] No unused imports or variables

---

## Performance Guidelines

### Backend
- **API response time (p95):** <200ms
- **DB query time (p95):** <100ms
- **Connection pooling:** Min 5, Max 20 per server
- **Caching:** Redis for frequently accessed data (agents, company config)

### Frontend
- **Initial page load:** <3s (Lighthouse 80+)
- **Component render:** Memoize expensive calculations (useMemo)
- **API calls:** Use React Query for deduplication, caching
- **Bundle size:** <250KB gzipped (core)

---

**Version:** 1.0
**Last Updated:** 2026-03-17
**Enforced By:** ESLint, TypeScript strict mode, Vitest
**See Also:** `./codebase-summary.md`, `./blueprint/` for architecture details.
