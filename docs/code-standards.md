# Code Standards & Codebase Structure

## Monorepo Organization

```
project-root/
├── apps/
│   ├── backend/               ← NestJS API + Scheduler
│   │   ├── src/
│   │   │   ├── shared/        ← @Global() module (repos, services, handlers)
│   │   │   ├── api/           ← HTTP controllers, DTOs
│   │   │   ├── scheduler/     ← Heartbeat timer
│   │   │   └── realtime/      ← WebSocket gateway
│   │   └── package.json
│   ├── web/                   ← React frontend (Vite)
│   │   ├── src/
│   │   │   ├── components/    ← React components
│   │   │   ├── pages/         ← Route pages
│   │   │   ├── hooks/         ← Custom React hooks
│   │   │   └── stores/        ← State management
│   │   └── package.json
│   └── executor/              ← Agent Executor (runs on Fly.io VM)
│       ├── src/
│       │   ├── executor/      ← HTTP server, request handler
│       │   ├── adapters/      ← Local adapter implementations
│       │   └── workspace/     ← File system, git management
│       └── package.json
├── packages/
│   ├── shared/                ← Shared types, constants, validators
│   │   ├── src/
│   │   │   ├── types/         ← TypeScript interfaces, enums
│   │   │   ├── constants/     ← Config values
│   │   │   ├── validators/    ← Zod schemas
│   │   │   └── utils/         ← Utility functions
│   │   └── package.json
│   ├── adapters/              ← Adapter registry (all 9+ adapters)
│   │   ├── src/
│   │   │   ├── claude/        ← Claude adapter
│   │   │   ├── codex/         ← Codex adapter
│   │   │   ├── gemini/        ← Gemini adapter
│   │   │   ├── opencode/      ← OpenCode adapter
│   │   │   ├── openclaw_gateway/
│   │   │   ├── pi/
│   │   │   ├── cursor/
│   │   │   ├── process/       ← Process executor
│   │   │   ├── http/          ← HTTP client
│   │   │   └── registry.ts    ← Central adapter registry
│   │   └── package.json
│   └── adapter-utils/         ← Shared utilities for adapters
│       └── src/
│           ├── session-codec/ ← Serialize/deserialize agent state
│           ├── stream-parser/ ← Parse streaming responses
│           └── utils/         ← Common functions
├── config/
│   ├── skills/                ← Agent instruction files (SKILL.md templates)
│   │   ├── ceo/
│   │   ├── engineer/
│   │   ├── designer/
│   │   └── ... (one per role)
│   └── templates/             ← Company setup templates
│       ├── saas-startup.json
│       ├── consulting.json
│       └── ...
├── tests/
│   ├── e2e/                   ← Playwright E2E tests
│   │   ├── signup.spec.ts
│   │   ├── heartbeat.spec.ts
│   │   └── ...
│   └── integration/           ← Integration tests
├── turbo.json                 ← Turborepo config
└── package.json               ← Root pnpm workspaces
```

## File Naming Conventions

| File Type | Convention | Example | Notes |
|-----------|-----------|---------|-------|
| TypeScript files | kebab-case | `execution-engine.ts` | Service, controller, handler files |
| React components | PascalCase | `TaskBoard.tsx` | React component files |
| Test files | `[name].spec.ts` | `agent-service.spec.ts` | Jest/Vitest test files |
| Database tables | snake_case | `heartbeat_runs` | SQL identifier |
| Enum values | UPPER_SNAKE_CASE | `AGENT_STATUS_ACTIVE` | TypeScript enums |
| Constants | UPPER_SNAKE_CASE | `MAX_AGENTS_PER_COMPANY` | Module-level constants |
| Type files | filename.types.ts | `agent.types.ts` | Exported types/interfaces |
| API DTOs | ClassName`Dto` | `CreateCompanyDto` | Request/response objects |
| Zod schemas | camelCase | `createCompanySchema` | Zod validation schemas |

## Clean Architecture Layers

All code is organized into four strict layers. Inner layers NEVER import from outer layers.

```
┌─────────────────────────────────────┐
│  Layer 4: Presentation              │
│  (Controllers, DTOs, Guards,        │
│   Interceptors, Filters, Pipes)     │
├─────────────────────────────────────┤
│  Layer 3: Application               │
│  (Commands, Queries, Services,      │
│   Event Handlers, Facades)          │
├─────────────────────────────────────┤
│  Layer 2: Infrastructure            │
│  (Drizzle Schemas, Repositories,    │
│   External Clients, Mappers)        │
├─────────────────────────────────────┤
│  Layer 1: Domain (NO FRAMEWORK)     │
│  (Entities, Enums, Interfaces,      │
│   Domain Exceptions)                │
└─────────────────────────────────────┘
```

### Domain Layer (src/domain/)
Pure TypeScript — **zero NestJS/Drizzle imports allowed**.

```typescript
// ✅ CORRECT
export interface ICompanyRepository {
  findById(id: string): Promise<Company | null>;
}

export class Company {
  constructor(public id: string, public name: string) {}
}

export class BudgetExceededException extends Error {}

export enum AgentStatus {
  ACTIVE = "active",
  PAUSED = "paused",
}

// ❌ WRONG - NestJS imports
import { Injectable } from "@nestjs/common";
import { Entity, PrimaryKey } from "drizzle-orm";
```

**Patterns:**
- Interfaces named `I{Entity}Repository` for data access contracts
- Classes for entities, domain logic, value objects
- Custom exceptions extending `Error`
- Enums for fixed sets of values
- No async/framework code

### Infrastructure Layer (src/infrastructure/)
Concrete implementations of domain interfaces. Owns database schemas.

```typescript
// ✅ CORRECT - Drizzle schema
export const companies = pgTable("companies", {
  id: uuid("id").primaryKey(),
  name: text("name").notNull(),
  companyId: uuid("company_id").notNull().references(() => companies.id),
});

// ✅ CORRECT - Repository implementation
@Injectable()
export class CompanyRepository implements ICompanyRepository {
  constructor(private db: Database) {}

  async findById(id: string): Promise<Company | null> {
    const row = await this.db.query.companies.findFirst({
      where: eq(companies.id, id),
    });
    return row ? this.toDomain(row) : null;
  }

  private toDomain(row: CompaniesRow): Company {
    return new Company(row.id, row.name);
  }
}

// ❌ WRONG - Mixing with application logic
export class CompanyRepository implements ICompanyRepository {
  async findById(id: string): Promise<Company | null> {
    // Should not contain business logic, only data access
    if (!isValidUuid(id)) throw new Error("Invalid ID");
    // ...
  }
}
```

**Patterns:**
- `@Injectable()` services for dependency injection
- Repository pattern for data access
- Mapper classes to convert between database rows and domain entities
- External service clients (Fly.io, S3, Redis)

### Application Layer (src/application/)
CQRS handlers, use cases, and cross-cutting services.

```typescript
// ✅ CORRECT - Command
export class CreateCompanyCommand {
  constructor(public userId: string, public name: string) {}
}

@CommandHandler(CreateCompanyCommand)
export class CreateCompanyHandler
  implements ICommandHandler<CreateCompanyCommand>
{
  constructor(
    private companyRepository: ICompanyRepository,
    private eventBus: EventBus,
  ) {}

  async execute(command: CreateCompanyCommand): Promise<string> {
    const company = new Company(generateUuid(), command.name);
    await this.companyRepository.save(company);
    this.eventBus.publish(new CompanyCreatedEvent(company.id));
    return company.id;
  }
}

// ✅ CORRECT - Query
export class GetCompanyQuery {
  constructor(public companyId: string) {}
}

export class GetCompanyDto {
  id: string;
  name: string;
}

@QueryHandler(GetCompanyQuery)
export class GetCompanyHandler implements IQueryHandler<GetCompanyQuery> {
  constructor(private companyRepository: ICompanyRepository) {}

  async execute(query: GetCompanyQuery): Promise<GetCompanyDto> {
    const company = await this.companyRepository.findById(query.companyId);
    if (!company) throw new CompanyNotFoundException();
    return { id: company.id, name: company.name };
  }
}

// ❌ WRONG - Business logic in controller, not command
@Post("companies")
async create(@Body() dto: CreateCompanyDto) {
  const company = new Company(...);
  await this.repository.save(company);
  return company;
}
```

**Patterns:**
- `Command` classes for mutations (immutable DTO)
- `CommandHandler` implements `ICommandHandler<T>`
- `Query` classes for reads
- `QueryHandler` implements `IQueryHandler<T>`
- Services implement interfaces, injected via constructor
- Event handlers for domain events
- No HTTP concerns (return DTOs, not responses)

### Presentation Layer (src/api/)
HTTP controllers, DTOs, guards, decorators.

```typescript
// ✅ CORRECT - Controller
@Controller("api/companies")
@UseGuards(BoardAuthGuard, CompanyAccessGuard)
export class CompanyController {
  constructor(
    private commandBus: CommandBus,
    private queryBus: QueryBus,
  ) {}

  @Post()
  async create(
    @CurrentActor() actor: IActor,
    @Body(new ZodValidationPipe(createCompanySchema))
    dto: CreateCompanyDto,
  ): Promise<GetCompanyDto> {
    const companyId = await this.commandBus.execute(
      new CreateCompanyCommand(actor.userId, dto.name),
    );
    return this.queryBus.execute(new GetCompanyQuery(companyId));
  }

  @Get(":companyId")
  @Roles("owner", "admin", "viewer")
  async getCompany(
    @Param("companyId") companyId: string,
  ): Promise<GetCompanyDto> {
    return this.queryBus.execute(new GetCompanyQuery(companyId));
  }
}

// ✅ CORRECT - DTO
export class CreateCompanyDto {
  name: string;
  description?: string;
}

export const createCompanySchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(1000).optional(),
});

// ❌ WRONG - Business logic in controller
@Post()
async create(@Body() dto: CreateCompanyDto) {
  if (!isValidCompanyName(dto.name)) throw new Error(...);
  const company = new Company(...);
  await this.db.save(company);
  return company;
}
```

**Patterns:**
- Controllers dispatch to `commandBus` or `queryBus`
- DTOs for request/response validation
- Zod schemas for input validation
- Guards for cross-cutting auth/authz
- Decorators for parameter extraction
- No business logic in controllers

## CQRS Pattern

### Commands (State Mutations)

```typescript
// Command definition
export class InvokeHeartbeatCommand {
  constructor(
    public companyId: string,
    public agentId: string,
  ) {}
}

// Command handler
@CommandHandler(InvokeHeartbeatCommand)
export class InvokeHeartbeatHandler
  implements ICommandHandler<InvokeHeartbeatCommand, InvokeHeartbeatResult>
{
  constructor(
    private executionEngine: IExecutionEngineService,
    private costRepository: ICostRepository,
    private agentRepository: IAgentRepository,
  ) {}

  async execute(command: InvokeHeartbeatCommand): Promise<InvokeHeartbeatResult> {
    // Retrieve context
    const agent = await this.agentRepository.findById(command.agentId);
    if (!agent) throw new AgentNotFoundException();

    // Execute
    const result = await this.executionEngine.execute({...});

    // Record cost
    await this.costRepository.recordEvent({...});

    return result;
  }
}

// Controller dispatch
@Post("companies/:companyId/agents/:agentId/heartbeat")
async invokeHeartbeat(
  @Param("companyId") companyId: string,
  @Param("agentId") agentId: string,
) {
  return this.commandBus.execute(
    new InvokeHeartbeatCommand(companyId, agentId),
  );
}
```

### Queries (Data Reads)

```typescript
export class GetAgentQuery {
  constructor(public companyId: string, public agentId: string) {}
}

export class GetAgentDto {
  id: string;
  name: string;
  status: string;
  budgetMonthlyCents: number;
  spentMonthlyCents: number;
}

@QueryHandler(GetAgentQuery)
export class GetAgentHandler implements IQueryHandler<GetAgentQuery, GetAgentDto> {
  constructor(private agentRepository: IAgentRepository) {}

  async execute(query: GetAgentQuery): Promise<GetAgentDto> {
    const agent = await this.agentRepository.findById(query.agentId);
    if (!agent) throw new AgentNotFoundException();
    return {
      id: agent.id,
      name: agent.name,
      status: agent.status,
      budgetMonthlyCents: agent.budgetMonthlyCents,
      spentMonthlyCents: agent.spentMonthlyCents,
    };
  }
}
```

## Validation with Zod

All request/response validation via Zod schemas. Apply validation via `ZodValidationPipe`.

```typescript
// Define schema
export const createAgentSchema = z.object({
  name: z.string().min(1).max(100),
  role: z.enum(["ceo", "engineer", "designer", "marketer"]),
  adapterType: z.string(),
  budgetMonthlyCents: z.number().positive().optional(),
});

export type CreateAgentDto = z.infer<typeof createAgentSchema>;

// Apply validation in controller
@Post("agents")
async createAgent(
  @Body(new ZodValidationPipe(createAgentSchema))
  dto: CreateAgentDto,
) {
  // dto is type-safe and validated
  return this.commandBus.execute(new CreateAgentCommand(dto));
}

// Error response
// 400 Bad Request
// {
//   "error": "Validation failed",
//   "details": {
//     "name": ["String must contain at least 1 character"],
//     "budgetMonthlyCents": ["Must be a positive number"]
//   }
// }
```

## Error Handling

### Domain Exceptions

```typescript
// Domain layer - no framework imports
export class CompanyNotFoundException extends Error {
  constructor(companyId: string) {
    super(`Company ${companyId} not found`);
    this.name = "CompanyNotFoundException";
  }
}

export class BudgetExceededException extends Error {
  constructor(agentId: string, limit: number, spent: number) {
    super(
      `Agent ${agentId} has exceeded budget: limit ${limit}, spent ${spent}`,
    );
    this.name = "BudgetExceededException";
  }
}

// Handler throws domain exception
async execute(command: InvokeHeartbeatCommand): Promise<any> {
  const agent = await this.agentRepository.findById(command.agentId);
  if (!agent) throw new AgentNotFoundException(command.agentId);
  if (agent.spentMonthlyCents >= agent.budgetMonthlyCents) {
    throw new BudgetExceededException(
      command.agentId,
      agent.budgetMonthlyCents,
      agent.spentMonthlyCents,
    );
  }
}
```

### Exception Filter

HTTP exception filter converts domain exceptions to HTTP responses:

```typescript
@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: Error, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    let statusCode = 500;
    let errorMessage = "Internal server error";

    if (exception instanceof CompanyNotFoundException) {
      statusCode = 404;
      errorMessage = exception.message;
    } else if (exception instanceof BudgetExceededException) {
      statusCode = 400;
      errorMessage = exception.message;
    } else if (exception instanceof BadRequestException) {
      statusCode = 400;
      errorMessage = exception.message;
    }

    response.status(statusCode).json({
      error: errorMessage,
      details: {
        timestamp: new Date().toISOString(),
        path: ctx.getRequest().url,
      },
    });
  }
}
```

## TypeScript Conventions

### Type Naming
- Interfaces: `I{Entity}` (e.g., `ICompany`, `IAgentRepository`)
- Classes: `{Entity}` (e.g., `Company`, `Agent`)
- Enums: `{Entity}Status` or similar (e.g., `AgentStatus`, `IssueStatus`)
- DTOs: `{Operation}{Entity}Dto` (e.g., `CreateCompanyDto`, `GetAgentDto`)
- Types: `{Entity}Type` or `{Entity}Props` (e.g., `CompanyType`, `CompanyProps`)

### Imports
- Always use `import` not `require`
- Order: external → internal → relative (3 groups)
- Remove unused imports

```typescript
// ✅ CORRECT
import { Injectable } from "@nestjs/common";
import { CommandHandler } from "@nestjs/cqrs";

import { ICompanyRepository } from "@/domain/repositories";
import type { Company } from "@/domain/entities";

import { CreateCompanyDto } from "./dto";
```

### Optional vs Required
Use `?` only when truly optional. Prefer explicit types.

```typescript
// ✅ CORRECT
export class CreateAgentDto {
  name: string;
  role: string;
  budgetMonthlyCents?: number; // Only if truly optional
}

// ❌ AVOID - too permissive
export class CreateAgentDto {
  name?: string;
  role?: string;
  [key: string]: any;
}
```

## Module Structure

All modules use dependency injection via constructor:

```typescript
// ✅ CORRECT
@Module({
  providers: [
    CompanyRepository,
    AgentRepository,
    CreateCompanyHandler,
    GetCompanyHandler,
  ],
  exports: [CompanyRepository, AgentRepository],
})
export class CompanyModule {}

// In service
@Injectable()
export class CreateCompanyHandler {
  constructor(private companyRepository: ICompanyRepository) {}
}

// ❌ WRONG - mixing instance creation with business logic
const repo = new CompanyRepository(db);
const result = repo.findById(id);
```

## Testing Patterns

### Unit Tests (Vitest)
Test handlers, services, utilities in isolation.

```typescript
describe("InvokeHeartbeatHandler", () => {
  let handler: InvokeHeartbeatHandler;
  let mockExecutionEngine: jest.Mock;
  let mockCostRepository: jest.Mock;

  beforeEach(() => {
    mockExecutionEngine = {
      execute: jest.fn(),
    };
    mockCostRepository = {
      recordEvent: jest.fn(),
    };
    handler = new InvokeHeartbeatHandler(
      mockExecutionEngine,
      mockCostRepository,
    );
  });

  it("should execute heartbeat and record cost", async () => {
    const command = new InvokeHeartbeatCommand("company1", "agent1");

    mockExecutionEngine.execute.mockResolvedValue({
      success: true,
      output: "Done",
    });

    const result = await handler.execute(command);

    expect(mockExecutionEngine.execute).toHaveBeenCalled();
    expect(mockCostRepository.recordEvent).toHaveBeenCalled();
    expect(result.success).toBe(true);
  });

  it("should throw if agent not found", async () => {
    const command = new InvokeHeartbeatCommand("company1", "invalid");
    // Setup mock to return null
    await expect(handler.execute(command)).rejects.toThrow(
      AgentNotFoundException,
    );
  });
});
```

### Integration Tests
Test with real database (test database):

```typescript
describe("CompanyRepository (Integration)", () => {
  let repository: CompanyRepository;
  let db: Database;

  beforeAll(async () => {
    db = await createTestDatabase();
  });

  afterAll(async () => {
    await db.close();
  });

  it("should save and retrieve company", async () => {
    const company = new Company("id1", "Test Co");
    await repository.save(company);

    const retrieved = await repository.findById("id1");
    expect(retrieved?.name).toBe("Test Co");
  });
});
```

### E2E Tests (Playwright)
Test full user journeys in browser:

```typescript
test("User can create company and hire agent", async ({ page }) => {
  await page.goto("/");
  await page.click("text=Sign up");
  await page.fill('input[name="email"]', "test@example.com");
  await page.fill('input[name="password"]', "secure123");
  await page.click('button:has-text("Create Account")');

  await page.waitForURL("/dashboard");
  await page.click("text=New Company");
  await page.fill('input[name="name"]', "My AI Company");
  await page.click("text=Create");

  await expect(page.locator("text=My AI Company")).toBeVisible();
});
```

## Dependency Injection

All dependencies injected via constructor. No global state or singletons.

```typescript
// ✅ CORRECT
@Injectable()
export class CreateCompanyHandler {
  constructor(
    private companyRepository: ICompanyRepository,
    private eventBus: EventBus,
    private encryptionService: IEncryptionService,
  ) {}
}

// ❌ WRONG - global or static
export class CreateCompanyHandler {
  constructor() {
    this.repo = getGlobalCompanyRepository(); // No globals
  }

  execute(command) {
    const encrypted = EncryptionService.encrypt(...); // No statics
  }
}
```

## Code Comments

Write comments for *why*, not *what*. Code should be self-explanatory.

```typescript
// ✅ CLEAR - explains decision
// We use atomic checkout (409 conflict) instead of locking to avoid deadlocks
// in multi-agent scenarios. Agent retries if conflict occurs.
async checkoutIssue(issueId: string, agentId: string): Promise<void> {
  const issue = await this.issueRepository.findById(issueId);
  if (issue.checkedOutBy && issue.checkedOutBy !== agentId) {
    throw new ConflictException("Issue already checked out");
  }
}

// ❌ UNCLEAR - describes what code does
// Set the checked out flag to the agent ID
issue.checkedOutBy = agentId;
```

## Performance Guidelines

1. **Pagination:** Always paginate list endpoints (limit 50, offset-based or cursor)
2. **Indexes:** Add indexes on frequently queried columns (companyId, status, createdAt)
3. **N+1 prevention:** Use JOIN or batch queries, not loops
4. **Caching:** Cache rarely-changing data (company config, agent templates)
5. **Connection pooling:** Use PgBouncer (Neon managed)

## Security Guidelines

1. **Input validation:** Zod on all inputs
2. **Encryption:** AES-256 for secrets at rest
3. **Audit logs:** All mutations logged to activity log
4. **Rate limiting:** 100 req/min per IP on public endpoints
5. **HTTPS:** Always, no exceptions
6. **CORS:** Explicitly whitelist origins

## Linting & Formatting

- **Linter:** ESLint (TBD specific rules)
- **Formatter:** Prettier (default config)
- **Pre-commit:** Lint and typecheck (via husky hook)
- **CI:** Typecheck, lint, test on every PR

Commands:
```bash
pnpm lint                  # Check linting
pnpm lint:fix              # Auto-fix
pnpm typecheck             # TypeScript check
pnpm format                # Format with Prettier
pnpm test                  # Run tests
```

All code must pass linting and typecheck before merge.
