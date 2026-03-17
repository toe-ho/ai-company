# Phase 7: Application Layer - Core Services

## Context Links
- [11 - Backend Architecture](../../docs/blueprint/03-architecture/11-backend-architecture.md) (service layer table)
- [09 - System Architecture](../../docs/blueprint/03-architecture/09-system-architecture.md) (execution engine)
- [23 - Config & Environment](../../docs/blueprint/06-infrastructure/23-config-and-environment.md)

## Overview
- **Priority:** P1
- **Status:** pending
- **Effort:** 2h
- **Description:** Implement 6 cross-cutting application services behind interfaces. These are injected into CQRS handlers.

## Key Insights
- Interface/impl separation enables easy mocking for tests
- ExecutionEngineService is the bridge between control plane and Fly.io VMs
- ApiKeyVaultService uses AES-256 encryption; ENCRYPTION_KEY from env
- RedisLiveEventsService uses Redis PUBLISH/SUBSCRIBE channels per company
- FlyioProvisionerService wraps Fly.io Machines REST API
- All external clients live in `infrastructure/external/`

## Requirements
### Functional
- ExecutionEngine: POST to VM executor + parse SSE stream
- Provisioner: create/start/stop/destroy Fly.io machines
- ApiKeyVault: AES-256 encrypt/decrypt/validate API keys
- LiveEvents: Redis pub/sub per company channel
- Storage: S3 put/get/delete/presign
- Encryption: AES-256 primitives

### Non-functional
- SSE parsing must handle partial chunks
- VM boot timeout: 30s max
- Encryption key: 32-byte from env

## Architecture
```
apps/backend/src/application/services/
├── interface/
│   ├── execution-engine-service.interface.ts
│   ├── provisioner-service.interface.ts
│   ├── api-key-vault-service.interface.ts
│   ├── live-events-service.interface.ts
│   ├── storage-service.interface.ts
│   └── encryption-service.interface.ts
└── impl/
    ├── execution-engine-service.ts
    ├── flyio-provisioner-service.ts
    ├── api-key-vault-service.ts
    ├── redis-live-events-service.ts
    ├── s3-storage-service.ts
    └── aes-encryption-service.ts

apps/backend/src/infrastructure/external/
├── flyio/
│   ├── flyio-client.ts
│   └── flyio-types.ts
├── redis/
│   ├── redis-module.ts
│   └── redis-client.ts
├── s3/
│   ├── s3-client.ts
│   └── s3-types.ts
└── stripe/
    └── stripe-client.ts          # Placeholder for future billing
```

## Related Code Files

### Files to Create
All files above (~18 files).

## Implementation Steps

1. **Create service interfaces** (`application/services/interface/`)
   ```typescript
   // execution-engine-service.interface.ts
   export interface IExecutionEngineService {
     execute(request: IExecutionRequest): AsyncIterable<IExecutionEvent>;
     cancel(runId: string, machineId: string): Promise<void>;
   }

   // provisioner-service.interface.ts
   export interface IProvisionerService {
     ensureMachine(companyId: string, config?: VmConfig): Promise<{ machineId: string; ip: string }>;
     hibernateMachine(companyId: string): Promise<void>;
     destroyMachine(companyId: string): Promise<void>;
     getMachineStatus(companyId: string): Promise<VmStatus>;
   }

   // api-key-vault-service.interface.ts
   export interface IApiKeyVaultService {
     store(companyId: string, provider: string, key: string, label?: string): Promise<void>;
     retrieve(companyId: string, provider: string): Promise<string>;
     validate(companyId: string, provider: string): Promise<boolean>;
     revoke(companyId: string, keyId: string): Promise<void>;
   }

   // live-events-service.interface.ts
   export interface ILiveEventsService {
     publish(companyId: string, event: { type: string; payload: unknown }): Promise<void>;
     subscribe(companyId: string, callback: (event: unknown) => void): () => void;
   }

   // storage-service.interface.ts
   export interface IStorageService {
     put(key: string, data: Buffer, contentType: string): Promise<string>;
     get(key: string): Promise<Buffer>;
     delete(key: string): Promise<void>;
     getSignedUrl(key: string, expiresIn?: number): Promise<string>;
   }

   // encryption-service.interface.ts
   export interface IEncryptionService {
     encrypt(plaintext: string): string;
     decrypt(ciphertext: string): string;
   }
   ```

2. **Create `AesEncryptionService`**
   - Uses `crypto.createCipheriv` / `createDecipheriv`
   - Algorithm: `aes-256-gcm`
   - ENCRYPTION_KEY from ConfigService (32 bytes, base64)
   - Random IV per encryption, prepended to ciphertext
   - Output format: `iv:authTag:ciphertext` (all base64)

3. **Create `ApiKeyVaultService`**
   - `store()`: encrypt key -> compute keyHash -> upsert to companyApiKeys table
   - `retrieve()`: find by (companyId, provider) -> decrypt -> return
   - `validate()`: retrieve key -> test against provider API (HEAD request)
   - `revoke()`: delete from table
   - Injects: IEncryptionService, ICompanyApiKeyRepository

4. **Create external clients**
   - `flyio-client.ts`: Wraps Fly.io Machines API (create, start, stop, delete, status)
     - Base URL: `https://api.machines.dev/v1`
     - Auth: Bearer token from FLY_API_TOKEN
   - `redis-client.ts`: ioredis connection from REDIS_URL, exports pub/sub clients
   - `redis-module.ts`: NestJS module providing Redis client
   - `s3-client.ts`: AWS SDK S3Client from S3_BUCKET, S3_REGION, S3_ENDPOINT

5. **Create `ExecutionEngineService`**
   <!-- Updated: Validation Session 1 - Local dev mode support -->
   - **Local dev mode:** When `FLY_API_TOKEN` is not set, post to `http://localhost:8080/execute` instead of Fly.io VM IP. This allows running apps/executor locally alongside backend.
   - `execute()`: POST to `http://${machineIp}:8080/execute` (or `localhost:8080` in dev) with execution request body
   - Parse SSE response using async generator:
     ```typescript
     async *execute(request: IExecutionRequest): AsyncIterable<IExecutionEvent> {
       const response = await fetch(url, { method: 'POST', body: JSON.stringify(request), headers: { 'Accept': 'text/event-stream' } });
       const reader = response.body.getReader();
       // Parse SSE: split by \n\n, extract event: and data: fields
       for await (const event of parseSSE(reader)) {
         yield event;
       }
     }
     ```
   - `cancel()`: POST to `http://${machineIp}:8080/cancel` with `{ runId }`

6. **Create `FlyioProvisionerService`**
   - `ensureMachine()`: check companyVms table -> if exists and stopped, start it -> if not exists, create new machine
   - Uses FlyioClient for API calls
   - Updates companyVms table with machine status
   - Wait for health check on :8080 (poll every 1s, timeout 30s)
   - `hibernateMachine()`: stop machine, update status
   - `destroyMachine()`: delete machine + volume, remove from table

7. **Create `RedisLiveEventsService`**
   - `publish()`: `redis.publish(\`company:${companyId}\`, JSON.stringify(event))`
   - `subscribe()`: `redis.subscribe(\`company:${companyId}\`)`, return unsubscribe function

8. **Create `S3StorageService`**
   - Standard S3 operations using AWS SDK v3
   - `put()` returns the object key
   - `getSignedUrl()` for presigned download URLs (default 1h expiry)

## Todo List
- [ ] 6 service interfaces
- [ ] AesEncryptionService
- [ ] ApiKeyVaultService
- [ ] FlyioClient + FlyioProvisionerService
- [ ] RedisClient + RedisModule + RedisLiveEventsService
- [ ] S3Client + S3StorageService
- [ ] ExecutionEngineService (SSE parser)
- [ ] Stripe placeholder
- [ ] typecheck passes

## Success Criteria
- All services implement their interfaces
- AES encryption roundtrip works (encrypt -> decrypt = original)
- SSE parser handles partial chunks correctly
- All external clients configurable via env vars
- `turbo typecheck` passes

## Risk Assessment
- **SSE parsing edge cases:** Partial data across chunks; test with mock streams
- **Fly.io API changes:** Pin to v1 API; document expected endpoints
- **Redis connection drops:** ioredis auto-reconnects; handle SUBSCRIBE re-registration

## Security Considerations
- ENCRYPTION_KEY must be 32 bytes; validate on startup
- API keys encrypted at rest, decrypted only during execution
- Fly.io API token scoped to VM management only
- Redis URL includes auth credentials
- S3 credentials via AWS_ACCESS_KEY_ID/AWS_SECRET_ACCESS_KEY or IAM role

## Next Steps
- Phase 8: CQRS handlers inject these services
- Phase 11: SharedModule registers interface->impl bindings
