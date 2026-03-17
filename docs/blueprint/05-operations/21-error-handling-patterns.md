# 21 — Error Handling Patterns

Error handling strategy for the AI Company Platform (NestJS + Drizzle ORM + CQRS).

## Global Exception Filter

All unhandled exceptions are caught by `HttpExceptionFilter` and formatted consistently:

```typescript
// filter/HttpExceptionFilter.ts
@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = 500;
    let error = 'Internal server error';
    let details: Record<string, unknown> | undefined;

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const body = exception.getResponse();
      error = typeof body === 'string' ? body : (body as any).message ?? error;
      details = typeof body === 'object' ? (body as any).details : undefined;
    } else if (exception instanceof IssueAlreadyCheckedOutException) {
      status = 409;
      error = exception.message;
      details = exception.details;
    } else if (exception instanceof AgentOverBudgetException) {
      status = 422;
      error = exception.message;
      details = { agentId: exception.agentId, budgetCents: exception.budgetCents };
    } else if (exception instanceof MissingApiKeyException) {
      status = 422;
      error = exception.message;
      details = { provider: exception.provider };
    } else if (exception instanceof VmBootFailedException) {
      status = 503;
      error = 'Agent VM failed to start';
      details = { vmId: exception.vmId, reason: exception.reason };
    }

    // Log 5xx errors
    if (status >= 500) {
      logger.error({ err: exception, path: request.url }, 'Unhandled exception');
    }

    response.status(status).json({ error, ...(details ? { details } : {}) });
  }
}
```

Register globally in `main.ts`:
```typescript
app.useGlobalFilters(new HttpExceptionFilter());
```

## Error Response Format

All errors return the same shape:

```json
{
  "error": "Human-readable message",
  "details": {
    "field": "specific info"
  }
}
```

The `details` field is optional — only present when there is actionable structured data.

## Domain Exception → HTTP Status Mapping

| Exception | HTTP Status | When |
|-----------|-------------|------|
| `IssueAlreadyCheckedOutException` | 409 Conflict | Agent tries to checkout a task already owned by another run |
| `AgentOverBudgetException` | 422 Unprocessable Entity | Heartbeat invocation blocked by budget enforcement |
| `MissingApiKeyException` | 422 Unprocessable Entity | Company has no API key stored for the required provider |
| `VmBootFailedException` | 503 Service Unavailable | Fly.io VM failed to start within timeout |
| `NotFoundException` (NestJS) | 404 Not Found | Entity not found or not accessible to actor |
| `UnauthorizedException` (NestJS) | 401 Unauthorized | Missing or invalid session/JWT/API key |
| `ForbiddenException` (NestJS) | 403 Forbidden | Actor authenticated but lacks role/permission |
| `BadRequestException` (NestJS) | 400 Bad Request | Malformed request |
| `ConflictException` (NestJS) | 409 Conflict | General conflict (used for checkout when domain exception not thrown yet) |
| `UnprocessableEntityException` (NestJS) | 422 Unprocessable Entity | Valid request, business logic rejects it |

## Domain Exception Definitions

```typescript
// domain/exceptions/IssueAlreadyCheckedOutException.ts
export class IssueAlreadyCheckedOutException extends Error {
  constructor(
    public readonly issueId: string,
    public readonly checkedOutByRunId: string,
  ) {
    super(`Issue ${issueId} is already checked out by run ${checkedOutByRunId}`);
    this.details = { issueId, checkedOutByRunId };
  }
  readonly details: Record<string, string>;
}

// domain/exceptions/AgentOverBudgetException.ts
export class AgentOverBudgetException extends Error {
  constructor(
    public readonly agentId: string,
    public readonly budgetCents: number,
    public readonly spentCents: number,
  ) {
    super(`Agent ${agentId} has exceeded monthly budget`);
  }
}

// domain/exceptions/MissingApiKeyException.ts
export class MissingApiKeyException extends Error {
  constructor(public readonly provider: string) {
    super(`No API key configured for provider: ${provider}`);
  }
}

// domain/exceptions/VmBootFailedException.ts
export class VmBootFailedException extends Error {
  constructor(
    public readonly vmId: string,
    public readonly reason: string,
  ) {
    super(`VM ${vmId} failed to boot: ${reason}`);
  }
}
```

## Validation Errors (Zod Pipe)

`ZodValidationPipe` catches Zod parse errors and returns 400 with field-level details:

```typescript
// pipe/ZodValidationPipe.ts
@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private readonly schema: ZodSchema) {}

  transform(value: unknown) {
    const result = this.schema.safeParse(value);
    if (!result.success) {
      throw new BadRequestException({
        message: 'Validation failed',
        details: result.error.flatten().fieldErrors,
      });
    }
    return result.data;
  }
}
```

Example validation error response:
```json
{
  "error": "Validation failed",
  "details": {
    "title": ["Required"],
    "status": ["Invalid enum value. Expected 'todo' | 'in_progress' | 'done'"]
  }
}
```

## Agent-Facing vs User-Facing Errors

Agents and board users receive different error detail levels:

**Agent-facing errors** (from `agent/` controllers): Include operational details useful for the agent to understand and retry.
```json
{
  "error": "Issue already checked out",
  "details": {
    "issueId": "issue-abc",
    "checkedOutByRunId": "run-xyz",
    "hint": "Try a different issue from your inbox"
  }
}
```

**User-facing errors** (from `board/` controllers): Friendly messages, no internal IDs.
```json
{
  "error": "This task is already being worked on by another agent"
}
```

The distinction is enforced by the controller layer — agent controllers pass through domain exception details; board controllers may rephrase.

## Activity Log on Errors

Failed operations are logged to the activity log for audit purposes. `ActivityLogInterceptor` catches errors before they propagate and writes a `failed` entry:

```typescript
// interceptor/ActivityLogInterceptor.ts
async intercept(context: ExecutionContext, next: CallHandler) {
  const actor = extractActor(context);
  try {
    const result = await lastValueFrom(next.handle());
    // Success — logged by handler via LogActivityCommand
    return result;
  } catch (err) {
    // Log failure to activity log
    await this.commandBus.execute(new LogActivityCommand({
      companyId: actor.companyId,
      actorType: actor.type,
      actorId: actor.userId ?? actor.agentId,
      action: `${getResourceName(context)}.failed`,
      entityType: getEntityType(context),
      errorMessage: err.message,
    }));
    throw err; // Re-throw for HttpExceptionFilter
  }
}
```

This ensures that even failed operations appear in the activity log, which is critical for debugging and governance.

## Heartbeat Run Error Handling

When a heartbeat run fails (VM crash, timeout, executor error), the run status is set to `failed` and the error is stored:

```typescript
// InvokeHeartbeatHandler — error handling section
try {
  for await (const event of executionEngine.execute(request)) {
    await heartbeatRunEventRepo.create({ runId, ...event });
    await liveEventsService.publish(companyId, event);
  }
  await heartbeatRunRepo.update(runId, { status: 'succeeded', finishedAt: new Date() });
} catch (err) {
  await heartbeatRunRepo.update(runId, {
    status: 'failed',
    finishedAt: new Date(),
    resultJson: { error: err.message, stack: err.stack },
  });

  await commandBus.execute(new LogActivityCommand({
    companyId,
    actorType: 'system',
    action: 'heartbeat.failed',
    entityType: 'heartbeat_run',
    entityId: runId,
    errorMessage: err.message,
  }));

  // Do NOT re-throw — heartbeat failures are non-fatal to the scheduler
}
```

Key design: heartbeat failures are **swallowed at the handler level**. The run record stores the failure details. The scheduler continues ticking. Individual agent failures do not crash the system.

For `VmBootFailedException` specifically, the provisioner retries once before marking the run as failed. After 3 consecutive VM boot failures for a company, the provisioner auto-pauses all agents for that company and notifies via live event.
