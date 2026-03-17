# Phase 11: Cross-cutting & NestJS Modules

## Context Links
- [12 - API Architecture](../../docs/blueprint/03-architecture/12-api-architecture-nestjs.md) (module/ section)
- [21 - Error Handling](../../docs/blueprint/05-operations/21-error-handling-patterns.md)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 2h
- **Description:** Create cross-cutting concerns (filters, pipes, interceptors) and NestJS module definitions. Wire everything together in AppModule + main.ts bootstrap.

## Key Insights
- `SharedModule` is `@Global()` -- repositories and services available everywhere
- `ApiModule` groups all HTTP controllers
- `SchedulerModule` handles heartbeat ticks
- `RealtimeModule` handles WebSocket gateway
- `AuthModule` provides guards (Phase 6)
- `HttpExceptionFilter` maps domain exceptions to HTTP statuses
- `ZodValidationPipe` validates DTOs per endpoint
- `ActivityLogInterceptor` auto-logs mutations

## Architecture
```
apps/backend/src/
├── module/
│   ├── shared.module.ts          # @Global, all repos + services + handlers
│   ├── api.module.ts             # All controllers
│   ├── scheduler.module.ts       # Heartbeat timer
│   └── realtime.module.ts        # WebSocket gateway
├── filter/
│   └── http-exception-filter.ts
├── pipe/
│   └── zod-validation-pipe.ts
├── interceptor/
│   ├── activity-log-interceptor.ts
│   ├── company-scope-interceptor.ts
│   └── http-logger-interceptor.ts
├── config/
│   ├── app-config.ts
│   ├── database-config.ts
│   ├── redis-config.ts
│   ├── flyio-config.ts
│   └── auth-config.ts
├── app.module.ts
└── main.ts
```

## Related Code Files

### Files to Create
All files above (~16 files).

## Implementation Steps

1. **Create config files** (`config/`)
   - `app-config.ts`: PORT, HOST, NODE_ENV parsed + validated with Zod
   - `database-config.ts`: DATABASE_URL
   - `redis-config.ts`: REDIS_URL
   - `flyio-config.ts`: FLY_API_TOKEN, FLY_APP_NAME, FLY_REGION, FLY_VM_SIZE, FLY_IDLE_TIMEOUT_MIN
   - `auth-config.ts`: AUTH_SECRET, AUTH_URL, AGENT_JWT_SECRET, AGENT_JWT_TTL_SECONDS

2. **Create `HttpExceptionFilter`**
   - `@Catch()` decorator catches all exceptions
   - Map domain exceptions to HTTP statuses per blueprint 21:
     - `IssueAlreadyCheckedOutException` -> 409
     - `AgentOverBudgetException` -> 422
     - `MissingApiKeyException` -> 422
     - `VmBootFailedException` -> 503
   - Standard NestJS exceptions pass through
   - Unknown exceptions -> 500 + log error
   - Response shape: `{ error: string, details?: object }`

3. **Create `ZodValidationPipe`**
   - Constructor takes a `ZodSchema`
   - `transform(value)`: safeParse -> if error throw BadRequestException with `error.flatten().fieldErrors`
   - Used per-endpoint: `@UsePipes(new ZodValidationPipe(CreateCompanySchema))`

4. **Create interceptors**
   - `HttpLoggerInterceptor`: Log request method/path/duration. Scrub Authorization headers.
   - `ActivityLogInterceptor`: On successful mutations (POST/PATCH/DELETE), dispatch `LogActivityCommand`. On failure, log error activity.
   - `CompanyScopeInterceptor`: Auto-inject companyId into query objects from route params.

5. **Create `SharedModule`**
   ```typescript
   @Global()
   @Module({
     imports: [DatabaseModule, CqrsModule, RedisModule],
     providers: [
       // Repositories (interface -> impl)
       { provide: 'ICompanyRepository', useClass: CompanyRepository },
       { provide: 'IAgentRepository', useClass: AgentRepository },
       // ... all 16+ repositories

       // Services (interface -> impl)
       { provide: 'IExecutionEngineService', useClass: ExecutionEngineService },
       { provide: 'IProvisionerService', useClass: FlyioProvisionerService },
       { provide: 'IApiKeyVaultService', useClass: ApiKeyVaultService },
       { provide: 'ILiveEventsService', useClass: RedisLiveEventsService },
       { provide: 'IStorageService', useClass: S3StorageService },
       { provide: 'IEncryptionService', useClass: AesEncryptionService },

       // CQRS handlers
       ...CommandHandlers,
       ...QueryHandlers,
       ...EventHandlers,
     ],
     exports: [/* all interface tokens */],
   })
   export class SharedModule {}
   ```

6. **Create `ApiModule`**
   - Imports: SharedModule (implicit via @Global)
   - Controllers: all board, agent, internal, public controllers (from Phase 12)
   - Placeholder until Phase 12

7. **Create `SchedulerModule`**
   - Imports: ScheduleModule.forRoot()
   - Providers: HeartbeatSchedulerService (Phase 13)
   - Placeholder until Phase 13

8. **Create `RealtimeModule`**
   - Providers: RealtimeGateway (Phase 13)
   - Placeholder until Phase 13

9. **Create `AppModule`**
   ```typescript
   @Module({
     imports: [
       ConfigModule.forRoot({ isGlobal: true }),
       SharedModule,
       AuthModule,
       ApiModule,
       SchedulerModule,
       RealtimeModule,
     ],
   })
   export class AppModule {}
   ```

10. **Create `main.ts`**
    ```typescript
    async function bootstrap() {
      const app = await NestFactory.create(AppModule, { rawBody: true });
      app.useGlobalFilters(new HttpExceptionFilter());
      app.useGlobalInterceptors(new HttpLoggerInterceptor());
      app.enableCors({ origin: process.env.AUTH_URL, credentials: true });

      const port = process.env.PORT ?? 3100;
      await app.listen(port, '0.0.0.0');
    }
    bootstrap();
    ```
    - `rawBody: true` needed for Better Auth body access
    - CORS with credentials for session cookies

## Todo List
- [ ] 5 config files
- [ ] HttpExceptionFilter
- [ ] ZodValidationPipe
- [ ] 3 interceptors
- [ ] SharedModule (register all repos + services + handlers)
- [ ] ApiModule (placeholder)
- [ ] SchedulerModule (placeholder)
- [ ] RealtimeModule (placeholder)
- [ ] AppModule
- [ ] main.ts bootstrap
- [ ] typecheck passes
- [ ] `turbo build` succeeds for backend

## Success Criteria
- `NestFactory.create(AppModule)` bootstraps without errors
- All providers properly registered and injectable
- Exception filter formats errors correctly
- Zod validation pipe returns field-level errors
- `turbo build --filter=@ai-company/backend` succeeds

## Risk Assessment
- **Circular deps:** SharedModule must not import ApiModule. Use @Global + exports.
- **Provider count:** ~80+ providers in SharedModule. NestJS handles this fine.
- **Raw body parsing:** Required by Better Auth; may conflict with some middleware

## Security Considerations
- HttpLoggerInterceptor scrubs Authorization headers from logs
- CORS restricted to AUTH_URL origin
- Global exception filter hides internal details for 500 errors

## Next Steps
- Phase 12: Controllers registered in ApiModule
- Phase 13: Scheduler and Realtime modules implemented
