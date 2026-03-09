# TypeScript Standards

> This file defines the specific standards for this boilerplate-api project.
> **Reference**: Always consult `CLAUDE.md` for project-specific instructions.

---

## Table of Contents

| # | Section | Description |
|---|---------|-------------|
| 1 | [Version](#version) | TypeScript and Bun versions |
| 2 | [Strict Configuration](#strict-configuration-mandatory) | tsconfig.json requirements |
| 3 | [Frameworks & Libraries](#frameworks--libraries) | Required packages |
| 4 | [Type Safety](#type-safety) | Never use `any`, use `unknown` with narrowing |
| 5 | [TypeBox Validation Patterns](#typebox-validation-patterns) | Schema validation with Elysia's `t` |
| 6 | [Dependency Injection](#dependency-injection) | tsyringe patterns |
| 7 | [AsyncLocalStorage for Context](#asynclocalstorage-for-context) | Request context propagation |
| 8 | [Testing](#testing) | Bun test runner, testcontainers, Eden client |
| 9 | [Error Handling](#error-handling) | Exception hierarchy with BaseException |
| 10 | [Function Design](#function-design-mandatory) | Single responsibility principle |
| 11 | [File Organization](#file-organization-mandatory) | File-level single responsibility |
| 12 | [Naming Conventions](#naming-conventions) | Files, interfaces, types |
| 13 | [Directory Structure](#directory-structure) | Clean Architecture module pattern |
| 14 | [Decorators](#decorators) | @trace, @safe, @injectable, @inject |
| 15 | [Database Patterns](#database-patterns) | Kysely query builder conventions |
| 16 | [Observability](#observability) | OpenTelemetry, Pino logging |
| 17 | [Route & OpenAPI Patterns](#route--openapi-patterns) | Elysia routes with error models |

**Meta-sections (not checked by agents):**
- [Checklist](#checklist) - Self-verification before submitting code

---

## Version

- TypeScript 5.0+ (ESNext target)
- Bun 1.3.10+
- Node.js compatibility via Bun runtime

---

## Strict Configuration (MANDATORY)

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "Preserve",
    "moduleResolution": "bundler",
    "lib": ["ESNext"],
    "strict": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "moduleDetection": "force",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true,
    "paths": {
      "@/*": ["src/*"],
      "@config/*": ["src/config/*"],
      "@shared/*": ["src/shared/*"],
      "@di/*": ["src/di/*"]
    }
  }
}
```

**Notes:**
- `experimentalDecorators` + `emitDecoratorMetadata` are required by tsyringe.
- `skipLibCheck: true` — type-check own code only, skip `.d.ts` files.
- `noUnusedLocals` and `noUnusedParameters` are **disabled** in tsconfig — Biome enforces these instead (`noUnusedImports: error`).

---

## Frameworks & Libraries

### Core Stack

| Library | Use Case |
|---------|----------|
| **Elysia** | HTTP framework with built-in TypeBox validation |
| **TypeBox** (via Elysia's `t`) | Runtime validation + OpenAPI schema generation |
| **tsyringe** | Dependency injection (constructor-based) |
| **Kysely** | Type-safe SQL query builder (not an ORM) |
| **pg** | PostgreSQL driver |
| **Pino** | Structured logging (pretty + OTLP transport) |
| **OpenTelemetry** | Distributed tracing and metrics |

### Testing

| Library | Use Case |
|---------|----------|
| `bun:test` | Built-in test runner (describe, test, expect) |
| testcontainers | Real Postgres/Redis containers for integration tests |
| `@elysiajs/eden` | Type-safe HTTP client for route testing |

### Dev Tooling

| Tool | Use Case |
|------|----------|
| Biome | Linting + formatting (replaces ESLint + Prettier) |
| lefthook | Pre-commit hooks (biome check + typecheck) |
| kysely-codegen | Auto-generate DB types from schema |

---

## Type Safety

### Never use `any`

```typescript
// ❌ FORBIDDEN
const data: any = fetchData();
function process(x: any) { ... }

// ✅ CORRECT - use unknown with type narrowing
const data: unknown = fetchData();
if (isUser(data)) {
    console.log(data.name);
}

// Type guard
function isUser(value: unknown): value is User {
    return (
        typeof value === 'object' &&
        value !== null &&
        'id' in value &&
        'name' in value
    );
}
```

---

## TypeBox Validation Patterns

Elysia's `t` (TypeBox) serves **dual purpose**: runtime validation and OpenAPI spec generation.

### Schema Definition

```typescript
import { t } from 'elysia';

// Reusable primitives
const EmailSchema = t.String({ format: 'email' });
const UuidSchema = t.String({ format: 'uuid' });

// Compose schemas for request/response
const CreateUserBody = t.Object({
    email: EmailSchema,
    name: t.String({ minLength: 1, maxLength: 100 }),
    role: t.Union([t.Literal('admin'), t.Literal('user')]),
});

const UserResponse = t.Object({
    id: UuidSchema,
    email: t.String(),
    name: t.String(),
    role: t.String(),
});
```

### Route with Validation

```typescript
import { errorModels } from '@shared/infra/http/models/errorModels.ts';

new Elysia()
    .use(errorModels)
    .post('/users', handler, {
        body: CreateUserBody,
        response: {
            201: UserResponse,
            422: 'ValidationResponse',    // TypeBox validation failures
            400: 'BadRequestResponse',    // Business validation
        },
    });
```

**Key rules:**
- Elysia auto-validates `body`/`params`/`query` against schemas — no manual validation needed.
- If validation fails, Elysia throws → global error handler converts to `ValidationException` → returns 422.
- Always register error models via `.use(errorModels)` before defining routes.
- Reference error schemas by **string name** in `response` object.

### Environment Validation

```typescript
import { Type } from '@sinclair/typebox';
import { parseSchema } from '@shared/infra/schema/parseSchema.ts';

const EnvSchema = Type.Object({
    PORT: Type.Number({ default: 3000 }),
    DATABASE_URL: Type.String(),
    NODE_ENV: Type.Union([
        Type.Literal('development'),
        Type.Literal('production'),
        Type.Literal('test'),
    ], { default: 'production' }),
});

export const env = parseSchema(EnvSchema, process.env);
```

All env vars are validated in `src/config/env.ts` (single source of truth).

---

## Dependency Injection

### Token Definition

```typescript
// src/modules/<name>/dependencies.ts
const USER_DEPENDENCIES = {
    UserRepository: Symbol.for('UserRepository'),
} as const;
```

- Use `Symbol.for()` with descriptive string keys.
- PascalCase token names matching the interface they represent.
- Merge module tokens into `src/di/dependencies.ts`.

### Container Registration

```typescript
// src/modules/<name>/container.ts
import { container } from 'tsyringe';

// Singleton: one instance for app lifetime
container.registerSingleton<UserRepository>(
    USER_DEPENDENCIES.UserRepository,
    KyselyUserRepository,
);

// Factory: new instance on each resolve (e.g., per-request logger)
container.register<LoggerProvider>(SHARED_DEPENDENCIES.LoggerProvider, {
    useFactory: () => PinoRequestLoggerProvider.create(pinoLogger),
});

// Value: use object directly
container.register(SHARED_DEPENDENCIES.DatabaseProvider, { useValue: kyselyDb });
```

- Side-effect import module `container.ts` from `src/di/container.ts`.
- Import order in `src/bootstrap.ts` is **critical**: `reflect-metadata` → tsyringe → DI container → Elysia app.

### Class Usage

```typescript
@injectable()
class KyselyUserRepository implements UserRepository {
    constructor(
        @inject(DEPENDENCIES.DatabaseProvider)
        private readonly db: Kysely<Database>,
        @inject(DEPENDENCIES.LoggerProvider)
        private readonly logger: LoggerProvider,
    ) {}
}
```

- Classes **must** have `@injectable()` decorator.
- Inject dependencies via constructor with `@inject(TOKEN)`.
- LoggerProvider is factory-registered — resolves per-request with AsyncLocalStorage context.

---

## AsyncLocalStorage for Context

### Store Definition

```typescript
// src/shared/infra/context/requestContext.ts
import { AsyncLocalStorage } from 'node:async_hooks';

type RequestStore = {
    requestId: string;
};

const requestContextStorage = new AsyncLocalStorage<RequestStore>();

export { requestContextStorage, type RequestStore };
```

### Elysia Hook (sets context per request)

```typescript
// src/shared/infra/http/hooks/requestContext.ts
const requestContext = new Elysia({ name: 'requestContext' })
    .guard({
        headers: t.Object({
            'x-request-id': t.Optional(t.String({ maxLength: 36 })),
        }),
    })
    .onRequest(function requestContextHandler({ request, set }) {
        const requestId = request.headers.get('x-request-id') ?? ulid();
        requestContextStorage.enterWith({ requestId });
        set.headers['x-request-id'] = requestId;
    })
    .as('scoped');
```

### Logger Interface & Implementation

```typescript
// src/shared/domain/providers/LoggerProvider/LoggerProvider.ts
interface LoggerProvider {
    info(message: string, meta?: Record<string, unknown>): void;
    warn(message: string, meta?: Record<string, unknown>): void;
    error(message: string | Error, meta?: Record<string, unknown>): void;
    debug(message: string, meta?: Record<string, unknown>): void;
    child(bindings: Record<string, unknown>): LoggerProvider;
    flush(): Promise<void>;
}
```

`PinoLoggerProvider` implements the interface with dual transport (pretty console + OTLP):

```typescript
// src/shared/infra/providers/LoggerProvider/PinoLoggerProvider.ts
@injectable()
class PinoLoggerProvider implements LoggerProvider {
    private logger!: Logger;

    constructor() {
        const prettyTransport = pino.transport({ target: 'pino-pretty', level: loggerConfig.level });
        const otelTransport = pino.transport({ target: 'pino-opentelemetry-transport', level: loggerConfig.level });
        const multistream = pino.multistream([
            { stream: prettyTransport, level: loggerConfig.level },
            { stream: otelTransport, level: loggerConfig.level },
        ]);
        this.logger = pino({ level: loggerConfig.level }, multistream);
    }

    info(message: string, meta?: Record<string, unknown>): void { this.logger.info(meta, message); }
    warn(message: string, meta?: Record<string, unknown>): void { this.logger.warn(meta, message); }
    error(message: string | Error, meta?: Record<string, unknown>): void { /* handles Error instances */ }
    debug(message: string, meta?: Record<string, unknown>): void { this.logger.debug(meta, message); }

    child(bindings: Record<string, unknown>): LoggerProvider {
        const child = Object.create(PinoLoggerProvider.prototype) as PinoLoggerProvider;
        child.logger = this.logger.child(bindings);
        return child;
    }
}
```

### Per-request Logger (via DI factory)

`PinoRequestLoggerProvider` creates a child logger enriched with `requestId` and `traceId`:

```typescript
// src/shared/infra/providers/LoggerProvider/PinoRequestLoggerProvider.ts
class PinoRequestLoggerProvider {
    static create(rootLogger: LoggerProvider): LoggerProvider {
        const store = requestContextStorage.getStore();
        const spanContext = getCurrentSpan()?.spanContext();

        return rootLogger.child({
            requestId: store?.requestId,
            traceId: spanContext?.traceId,
        });
    }
}
```

Registered as a DI factory so each resolve gets a fresh child logger with the current request's context:

```typescript
container.register<LoggerProvider>(SHARED_DEPENDENCIES.LoggerProvider, {
    useFactory: () => PinoRequestLoggerProvider.create(pinoLogger),
});
```

### Trace Decorator (`@trace()`)

Wraps any method in an OpenTelemetry span via `record()`:

```typescript
// src/shared/infra/decorators/trace.ts
function trace(spanName?: string): MethodDecorator {
    return (_, propertyKey, descriptor: PropertyDescriptor) => {
        const originalMethod = descriptor.value;
        const methodName = String(propertyKey);

        descriptor.value = function (this: object, ...args: unknown[]) {
            const name = spanName ?? `${getClassName(this)}.${methodName}`;
            return record(name, () => originalMethod.apply(this, args));
        };

        return descriptor;
    };
}
```

- Without arguments: span named `ClassName.methodName` automatically.
- With argument: `@trace('custom-span-name')` for explicit naming.

### Context Span Processor

Propagates `requestId` from AsyncLocalStorage into every OTel span as an attribute:

```typescript
// src/shared/infra/processors/opentelemetry/contextSpanProcessor.ts
const attributeMapping = {
    requestId: 'http.request.header.x_request_id',
} satisfies Record<keyof RequestStore, string>;

const contextSpanProcessor: SpanProcessor = {
    onStart(span: Span) {
        const store = requestContextStorage.getStore();
        if (!store) return;

        for (const key in attributeMapping) {
            const storeKey = key as keyof typeof attributeMapping;
            const value = store[storeKey];
            if (value) {
                span.setAttribute(attributeMapping[storeKey], String(value));
            }
        }
    },
    // ...
};
```

- Accepts `x-request-id` header from clients or auto-generates a ULID for request correlation.
- Every span and log line carries the same `requestId`, enabling end-to-end tracing.

---

## Testing

### Test Framework

Bun's built-in test runner (`bun:test`):

```bash
bun test                               # All tests
bun test .unit.test                                      # Unit only
bun test --timeout 30000 .integration.test  # Integration
```

### File Naming

| Type | Pattern | Example |
|------|---------|---------|
| Unit | `<ClassName>.unit.test.ts` | `CheckHealthUseCase.unit.test.ts` |
| Integration | `<feature>.integration.test.ts` | `healthRoutes.integration.test.ts` |

Tests live **co-located** next to the source file they test.

### Unit Test Pattern

```typescript
import { describe, test, expect } from 'bun:test';
import '@shared/test/mocks.ts'; // Mock OTel before imports

import { CheckHealthUseCase } from './CheckHealthUseCase.ts';
import { HealthStatus } from '../../domain/enums/HealthStatus.ts';

describe('CheckHealthUseCase', () => {
    test('returns healthy status', () => {
        // Arrange
        const useCase = new CheckHealthUseCase();

        // Act
        const result = useCase.execute();

        // Assert
        expect(result.status).toBe(HealthStatus.Healthy);
    });
});
```

**Key rules:**
- Import `@shared/test/mocks.ts` **first** to mock `@elysiajs/opentelemetry`.
- No DI container in unit tests — instantiate directly with mocks.
- AAA pattern (Arrange-Act-Assert).

### Type-Safe Mocks

```typescript
import { mock } from 'bun:test';

// Mock external modules
mock.module('@elysiajs/opentelemetry', () => ({
    record: (_name: string, fn: () => unknown) => fn(),
}));

// Mock interfaces
const mockUserRepository: UserRepository = {
    findById: mock(() => Promise.resolve(null)),
    save: mock(() => Promise.resolve()),
};
```

### Integration Test Pattern

```typescript
import { describe, test, expect, beforeAll, afterAll } from 'bun:test';
import '@shared/test/mocks.ts';

import { testInfra } from '@shared/test/containers/index.ts';
import { createTreatyClient, type TreatyClient } from '@shared/test/treaty.ts';
import type { app } from '@/bootstrap.ts';

let client: TreatyClient<typeof app>;

beforeAll(async () => {
    await testInfra.setup();                               // Start Postgres + Redis
    const { app } = await import('@/bootstrap.ts');
    client = createTreatyClient(app);                      // Eden type-safe client
}, 60_000);

afterAll(async () => {
    await testInfra.teardown();
});

describe('Health Routes', () => {
    test('GET /healthz returns 200 with healthy status', async () => {
        const response = await client.healthz.get();
        expect(response.status).toBe(200);
        expect(response.data).toEqual({ status: 'healthy' });
    });
});
```

**Key rules:**
- Use testcontainers for real Postgres/Redis — no mocking infrastructure in integration tests.
- Long `beforeAll` timeout (60s) for container startup.
- Eden client (`@elysiajs/eden`) for type-safe HTTP assertions.
- Always clean up in `afterAll()`.

### Edge Case Coverage (MANDATORY)

Every test suite MUST include edge cases beyond the happy path.

| AC Type | Required Edge Cases | Minimum Count |
|---------|---------------------|---------------|
| Input validation | null, empty string, boundary values, invalid format | 3+ |
| CRUD operations | not found, duplicate, large payload | 3+ |
| Business logic | zero, negative, boundary conditions, invalid state | 3+ |
| Error handling | timeout, connection refused, service unavailable | 2+ |

```typescript
describe('CheckReadinessUseCase', () => {
    // Happy path
    test('returns healthy when all dependencies are up', async () => { ... });

    // Edge cases (MANDATORY)
    test('returns unhealthy when database is down', async () => { ... });
    test('returns unhealthy when cache is down', async () => { ... });
    test('returns unhealthy when both are down', async () => { ... });
});
```

---

## Error Handling

### Exception Hierarchy

```
BaseException (abstract)
├── BadRequestException          (400)
├── UnauthorizedException        (401)
├── ForbiddenException           (403)
├── NotFoundException            (404)
├── ValidationException          (422, supports `details` field)
├── InternalException            (500, reportable: true)
└── ServiceUnavailableException  (503, reportable: true)
```

All exceptions live in `src/shared/domain/exceptions/`.

### Usage

```typescript
// Throw from use cases or domain code
throw new NotFoundException('User not found');
throw new ValidationException('Invalid data', metadata, { email: 'Invalid format' });
throw new ServiceUnavailableException('Database unavailable');

// InternalException is auto-reportable (logged by error handler)
throw new InternalException({ operation: 'createUser', input });
```

### Global Error Handler

The `errorHandler` hook (`src/shared/infra/hooks/errorHandler.ts`) automatically:
1. Catches all exceptions.
2. Converts Elysia-specific errors (validation, parse, not found) to domain exceptions via `ExceptionMapperChain`.
3. Logs reportable exceptions (`reportable: true`).
4. Maps to HTTP status via `ExceptionHttpMapper`.
5. Returns `{ status, error, message }` (or `{ status, error, message, details }` for `ValidationException`).

### Exception Response Mapping in Routes

Every route **must** document its possible error responses:

```typescript
.get('/users/:id', handler, {
    params: t.Object({ id: t.String() }),
    response: {
        200: UserResponse,
        404: 'NotFoundResponse',              // Use case throws NotFoundException
        422: 'ValidationResponse',            // Params validation
    },
})
```

**Rules:**
- Routes with `body`/`params`/`query` schemas → add `422: 'ValidationResponse'`.
- Route-specific exceptions → map the corresponding status code.
- Use `.use(errorModels)` from `@shared/infra/http/models/errorModels.ts`.

---

## Function Design (MANDATORY)

**Single Responsibility Principle (SRP):** Each function MUST have exactly ONE responsibility.

| Rule | Description |
|------|-------------|
| **One responsibility per function** | A function should do ONE thing and do it well |
| **Max 20-30 lines** | If longer, break into smaller functions |
| **One level of abstraction** | Don't mix high-level and low-level operations |
| **Descriptive names** | Function name should describe its single responsibility |

```typescript
// ❌ BAD - Multiple responsibilities
async function processOrder(order: Order): Promise<void> {
    if (!order.items?.length) throw new Error('no items');
    let total = 0;
    for (const item of order.items) { total += item.price * item.quantity; }
    if (order.couponCode) { total = total * 0.9; }
    await db.orders.save(order);
    await sendEmail(order.customerEmail, 'Order confirmed');
}

// ✅ GOOD - Single responsibility per function
async function processOrder(order: Order): Promise<void> {
    validateOrder(order);
    const total = calculateTotal(order.items);
    const finalTotal = applyDiscount(total, order.couponCode);
    await saveOrder(order, finalTotal);
    await notifyCustomer(order.customerEmail);
}
```

### Signs a Function Has Multiple Responsibilities

| Sign | Action |
|------|--------|
| Multiple `// section` comments | Split at comment boundaries |
| "and" in function name | Split into separate functions |
| More than 3 parameters | Consider parameter object or splitting |
| Nested conditionals > 2 levels | Extract inner logic to functions |

---

## File Organization (MANDATORY)

**Single Responsibility per File:** Each file MUST represent ONE cohesive concept.

| Rule | Description |
|------|-------------|
| **One concept per file** | A file groups functions/types for a single domain concept |
| **Max 200-300 lines** | If longer, split by responsibility boundaries |
| **One class per file** | Each class gets its own file |
| **File name = content** | `HealthController.ts` MUST only contain the health controller |

### Signs a File Needs Splitting

| Sign | Action |
|------|--------|
| File exceeds 300 lines | Split at responsibility boundaries |
| Multiple exported classes | One class per file |
| Mix of unrelated concerns | Separate into dedicated files |
| More than 5 unrelated exports | Group related exports into separate modules |

---

## Naming Conventions

### File Naming

| Element | Convention | Example |
|---------|------------|---------|
| Classes / Components | PascalCase | `HealthController.ts` |
| Repository implementations | `Kysely<Feature>Repository` | `KyselyHealthRepository.ts` |
| Routes | `<feature>Routes.ts` | `healthRoutes.ts` |
| Schemas | `<feature>Schemas.ts` | `healthSchemas.ts` |
| DTOs | `<UseCaseName>DTO.ts` | `CheckReadinessUseCaseDTO.ts` |
| Enums | PascalCase | `HealthStatus.ts` |
| Config files | camelCase | `database.ts`, `logger.ts` |
| Unit tests | `<ClassName>.unit.test.ts` | `CheckHealthUseCase.unit.test.ts` |
| Integration tests | `<feature>.integration.test.ts` | `healthRoutes.integration.test.ts` |

### Code Naming

| Element | Convention | Example |
|---------|------------|---------|
| Interfaces | PascalCase (no `I` prefix) | `HealthRepository`, `LoggerProvider` |
| Types | PascalCase | `CreateUserInput` |
| Classes | PascalCase | `CheckHealthUseCase` |
| Functions / methods | camelCase | `checkHealth()`, `execute()` |
| Constants | UPPER_SNAKE | `MAX_RETRY_COUNT` |
| Enums | PascalCase + PascalCase values | `HealthStatus.Healthy` |
| DI tokens | PascalCase via `Symbol.for()` | `Symbol.for('HealthRepository')` |
| Variables | camelCase, descriptive | `healthRepository`, `cacheProvider` |

### Class Naming Patterns

| Type | Pattern | Example |
|------|---------|---------|
| Use cases | `<Action><Feature>UseCase` | `CheckHealthUseCase` |
| Controllers | `<Feature>Controller` | `HealthController` |
| Repositories (interface) | `<Feature>Repository` | `HealthRepository` |
| Repositories (impl) | `Kysely<Feature>Repository` | `KyselyHealthRepository` |
| Providers | `<Framework><Type>Provider` | `PinoLoggerProvider`, `RedisCacheProvider` |
| Exceptions | `<Type>Exception` | `NotFoundException` |
| Mappers | `<Source>Mapper` | `ExceptionHttpMapper` |
| Resolvers | `<Feature>Resolver` | `HealthStatusResolver` |

### Method Naming

| Context | Convention | Example |
|---------|------------|---------|
| Use case entry point | `execute()` | Always `execute()` |
| Repository CRUD | `add()`, `getById()`, `updateById()`, `deleteById()` | From GenericRepository |
| Health checks | `ping()` | Returns `Promise<boolean>` |
| Controller actions | Descriptive verbs | `checkHealth()`, `checkReadiness()` |

---

## Directory Structure

Clean Architecture module pattern:

```
src/
├── main.ts                        # Entry point
├── bootstrap.ts                   # Elysia app creation + plugins
├── instrumentation.ts             # OTel SDK (preloaded via bunfig.toml)
├── routes.ts                      # Root route registration
│
├── config/                        # Environment & configuration
│   ├── env.ts                     # Validated env vars (source of truth)
│   ├── app.ts
│   ├── database.ts
│   ├── cache.ts
│   ├── logger.ts
│   ├── openapi.ts
│   └── opentelemetry.ts
│
├── di/                            # DI setup
│   ├── dependencies.ts            # Merged token definitions
│   └── container.ts               # Side-effect imports of all module containers
│
├── shared/                        # Cross-cutting infrastructure
│   ├── domain/
│   │   ├── enums/                 # ExceptionCode, etc.
│   │   ├── exceptions/            # BaseException + concrete exceptions
│   │   ├── repositories/          # GenericRepository interface
│   │   ├── providers/             # LoggerProvider, CacheProvider interfaces
│   │   └── mappers/               # ErrorExceptionMapper interface
│   ├── infra/
│   │   ├── decorators/            # @trace(), @safe()
│   │   ├── providers/             # Pino, Redis implementations
│   │   ├── persistence/kysely/    # KyselyGenericRepository
│   │   ├── http/
│   │   │   ├── controllers/
│   │   │   ├── schemas/
│   │   │   ├── models/            # errorModels.ts
│   │   │   ├── hooks/             # errorHandler, requestContext, httpMetrics
│   │   │   └── mappers/           # ExceptionHttpMapper, ExceptionMapperChain
│   │   ├── context/               # AsyncLocalStorage requestContext
│   │   └── functions/             # Utility functions (once, parse, time)
│   ├── container.ts               # Shared DI registrations
│   ├── dependencies.ts            # Shared DI tokens
│   └── test/                      # Test utilities
│       ├── preload.ts             # Test env defaults
│       ├── mocks.ts               # OTel mock
│       ├── containers/            # testcontainers setup
│       ├── treaty.ts              # Eden client helper
│       └── createTestDB.ts        # Test Kysely instance
│
├── modules/                       # Feature modules
│   └── <name>/
│       ├── dependencies.ts        # Module DI tokens
│       ├── container.ts           # Module DI registrations
│       ├── domain/
│       │   ├── enums/
│       │   ├── repositories/      # Interfaces only
│       │   └── resolvers/         # Static business logic
│       ├── application/
│       │   └── usecases/
│       │       └── DTOs/
│       └── infra/
│           ├── http/
│           │   ├── controllers/
│           │   ├── schemas/
│           │   └── <name>Routes.ts
│           └── persistence/
│               └── kysely/
│
└── migrations/                    # Kysely migrations
```

**Key rules:**
- `domain/` = Pure interfaces, no framework imports.
- `application/` = Use cases depend only on domain interfaces.
- `infra/` = Concrete implementations wired via DI.
- Each module has its own `dependencies.ts` + `container.ts`.
- Use `health` module as the **reference implementation** for new modules.

---

## Decorators

### @trace() — OpenTelemetry Spans

```typescript
import { trace } from '@shared/infra/decorators/trace.ts';

@trace()  // Span named "HealthController.checkHealth"
async checkHealth(): Promise<HealthOutput> { ... }

@trace('custom-span-name')  // Custom span name
async execute(): Promise<Output> { ... }
```

- Apply to **all** controller methods, use cases, and repository methods.
- Wraps execution in OTel `record()`.

### @safe(fallback) — Error Swallowing

```typescript
import { safe } from '@shared/infra/decorators/safe.ts';

@trace()
@safe(false)  // Returns false on any error
async ping(): Promise<boolean> {
    await sql`SELECT 1`.execute(this.db);
    return true;
}
```

- Wraps method in try/catch, returns fallback value on error.
- Use for health checks, optional operations.

### @injectable() + @inject() — DI

```typescript
import { injectable, inject } from 'tsyringe';

@injectable()
class MyService {
    constructor(
        @inject(DEPENDENCIES.MyRepository)
        private readonly repo: MyRepository,
    ) {}
}
```

- `@injectable()` is **required** on any class resolved from the container.
- `@inject(TOKEN)` marks constructor parameters for DI resolution.

---

## Database Patterns

### Kysely Query Builder

```typescript
// Insert
await this.db
    .insertInto(this.tableName)
    .values(entity)
    .returningAll()
    .executeTakeFirstOrThrow();

// Select
await this.db
    .selectFrom(this.tableName)
    .where('id', '=', id)
    .selectAll()
    .executeTakeFirst();

// Update
await this.db
    .updateTable(this.tableName)
    .set(entity)
    .where('id', '=', id)
    .returningAll()
    .executeTakeFirstOrThrow();

// Delete
await this.db
    .deleteFrom(this.tableName)
    .where('id', '=', id)
    .execute();

// Raw SQL
await sql`SELECT 1`.execute(this.db);
```

### Generic Repository

Extend `KyselyGenericRepository` for standard CRUD or implement domain interface directly for custom operations.

```typescript
// Standard CRUD via generic repository
@injectable()
class KyselyUserRepository
    extends KyselyGenericRepository<'users', string>
    implements UserRepository
{
    constructor(@inject(DEPENDENCIES.DatabaseProvider) db: Kysely<Database>) {
        super('users', db);
    }
}

// Custom repository (domain interface only)
@injectable()
class KyselyHealthRepository implements HealthRepository {
    constructor(
        @inject(DEPENDENCIES.DatabaseProvider)
        private readonly db: Kysely<Database>,
    ) {}

    @trace()
    @safe(false)
    async ping(): Promise<boolean> {
        await sql`SELECT 1`.execute(this.db);
        return true;
    }
}
```

### Migrations

```bash
bun run db:migrate        # Run pending migrations
bun run db:migrate:down   # Rollback last migration
bun run db:migrate:make   # Create new migration file
bun run db:codegen        # Regenerate types → src/shared/infra/persistence/types.ts
```

- Migration files in `src/migrations/`.
- Always run `db:codegen` after schema changes to keep types in sync.

---

## Observability

### Logging

```typescript
// Resolve from DI (factory-created per request)
const logger = container.resolve<LoggerProvider>(DEPENDENCIES.LoggerProvider);

logger.info('Processing order', { orderId, userId });
logger.error('Failed to create user', { error, input });
logger.debug('Cache hit', { key });
logger.warn('Slow query detected', { duration });
```

**Rules:**
- **Never** use `console.log` — Biome warns on `noConsole`.
- Resolve `LoggerProvider` from DI — it includes `requestId` and `traceId` automatically.
- Use child loggers for additional context: `logger.child({ module: 'payments' })`.

### Tracing

All methods across controllers, use cases, and repositories **must** be decorated with `@trace()`:

```typescript
@trace()
async execute(input: CreateUserInput): Promise<User> { ... }
```

Spans are auto-named as `ClassName.methodName` and exported via OTel.

---

## Route & OpenAPI Patterns

### Route Definition

```typescript
import Elysia from 'elysia';
import { errorModels } from '@shared/infra/http/models/errorModels.ts';

const userRoutes = new Elysia({ prefix: '/users' })
    .use(errorModels)
    .get('/:id', ({ params }) => controller.getUser(params.id), {
        params: t.Object({ id: t.String() }),
        response: {
            200: UserResponse,
            404: 'NotFoundResponse',
            422: 'ValidationResponse',
        },
    })
    .post('/', ({ body }) => controller.createUser(body), {
        body: CreateUserBody,
        response: {
            201: UserResponse,
            400: 'BadRequestResponse',
            422: 'ValidationResponse',
        },
    });
```

### Error Response Mapping Rules

| Route has... | Must include |
|-------------|-------------|
| `body` / `params` / `query` (TypeBox schemas) | `422: 'ValidationResponse'` |
| Use case throws `NotFoundException` | `404: 'NotFoundResponse'` |
| Use case throws `BadRequestException` | `400: 'BadRequestResponse'` |
| Use case throws `ServiceUnavailableException` | `503: 'ServiceUnavailableResponse'` |
| Any route | Consider all exceptions the handler chain can throw |

### Controller Pattern

Controllers resolve use cases from the DI container and delegate:

```typescript
@injectable()
class UserController {
    constructor(
        @inject(DEPENDENCIES.LoggerProvider)
        private readonly logger: LoggerProvider,
    ) {}

    @trace()
    async getUser(id: string): Promise<UserOutput> {
        const useCase = container.resolve(GetUserUseCase);
        return useCase.execute({ id });
    }
}
```

---

## Biome Rules

| Rule | Value | Notes |
|------|-------|-------|
| Indent | 2 spaces | |
| Line width | 120 | |
| Quotes | Single | |
| Semicolons | Always | |
| Trailing commas | All | |
| Arrow parens | As needed | `x => x` not `(x) => x` |
| `noUnusedImports` | error | |
| `noConsole` | warn | Use LoggerProvider |
| `noDoubleEquals` | error | Use `===` |
| `useConst` | error | Prefer `const` |
| `noStaticOnlyClass` | off | Allow utility classes |

```bash
bun run check       # Lint + format check
bun run check:fix   # Lint + format auto-fix
```

---

## Checklist

Before submitting code, verify:

- [ ] `bun run typecheck` passes
- [ ] `bun run check` passes (Biome lint + format)
- [ ] No `any` types — use `unknown` with type narrowing
- [ ] No `console.log` — use injected LoggerProvider
- [ ] All public methods on controllers/use cases/repositories have `@trace()`
- [ ] New classes are `@injectable()` with `@inject()` constructor params
- [ ] DI tokens defined in `dependencies.ts`, registered in `container.ts`
- [ ] Routes document all possible error responses in `response` object
- [ ] Routes use `.use(errorModels)` before defining endpoints
- [ ] Domain layer has no framework imports (Elysia, Kysely, etc.)
- [ ] Unit tests import `@shared/test/mocks.ts` first
- [ ] Integration tests use testcontainers, not mocked infrastructure
- [ ] Edge cases covered (minimum 3 per test suite)
- [ ] Files under 300 lines, functions under 30 lines
- [ ] Import order in `bootstrap.ts` preserved (`reflect-metadata` first)
