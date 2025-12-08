# Effect Layers Deep Dive

**Advanced Dependency Injection with Effect-TS Layers**

This guide provides an in-depth exploration of Effect's Layer system for dependency injection, going beyond the basics covered in Module 3 of the main course.

---

## Table of Contents

1. [Layer Fundamentals](#layer-fundamentals)
2. [Layer Construction Patterns](#layer-construction-patterns)
3. [Layer Composition](#layer-composition)
4. [Advanced Layer Patterns](#advanced-layer-patterns)
5. [Managing Layer Lifecycles](#managing-layer-lifecycles)
6. [Testing Strategies](#testing-strategies)
7. [Real-World Architecture](#real-world-architecture)
8. [Performance Considerations](#performance-considerations)

---

## Layer Fundamentals

### Understanding Layers

A `Layer` is a blueprint for constructing services with their dependencies. Think of layers as:
- **Service factories** that know how to build services
- **Dependency graphs** that wire services together
- **Reusable configurations** that can be swapped (dev/test/prod)

```typescript
import { Effect, Context, Layer } from "effect"

// A Layer<ServiceOut, Error, RequirementsIn>
type Layer<Out, E, In> = {
  // Produces services of type Out
  // May fail with error type E
  // Requires services of type In
}
```

### The Three Ways to Create Layers

```typescript
import { Effect, Context, Layer } from "effect"

interface Logger {
  readonly log: (message: string) => Effect.Effect<void>
}

class Logger extends Context.Tag("Logger")<Logger, Logger>() {}

// 1. Layer.succeed - For services without dependencies
const LoggerLive = Layer.succeed(
  Logger,
  {
    log: (message) => Console.log(message)
  }
)
// Type: Layer<Logger, never, never>

// 2. Layer.effect - For services that need setup (Effect)
const LoggerWithSetup = Layer.effect(
  Logger,
  pipe(
    Effect.sync(() => console.log("Logger initialized")),
    Effect.map(() => ({
      log: (message) => Console.log(`[LOG] ${message}`)
    }))
  )
)
// Type: Layer<Logger, never, never>

// 3. Layer.scoped - For services that need cleanup
const LoggerWithCleanup = Layer.scoped(
  Logger,
  pipe(
    Effect.sync(() => openLogFile()),
    Effect.flatMap(logFile =>
      pipe(
        Effect.addFinalizer(() =>
          Effect.sync(() => logFile.close())
        ),
        Effect.map(() => ({
          log: (message) =>
            Effect.sync(() => logFile.write(message))
        }))
      )
    )
  )
)
// Type: Layer<Logger, never, Scope>
```

---

[↑ Back to Top](#table-of-contents)

## Layer Construction Patterns

### Pattern 1: Simple Service (No Dependencies)

```typescript
interface Config {
  readonly apiUrl: string
  readonly timeout: number
}

class Config extends Context.Tag("Config")<Config, Config>() {}

// Environment-based config
const makeConfig = (): Config => ({
  apiUrl: process.env.API_URL || "http://localhost:3000",
  timeout: parseInt(process.env.TIMEOUT || "5000", 10)
})

const ConfigLive = Layer.succeed(Config, makeConfig())

// Or with validation
const ConfigLiveValidated = Layer.effect(
  Config,
  pipe(
    Effect.sync(() => process.env.API_URL),
    Effect.flatMap(apiUrl =>
      !apiUrl
        ? Effect.fail({ _tag: "MissingConfig" as const, key: "API_URL" })
        : Effect.succeed({
            apiUrl,
            timeout: parseInt(process.env.TIMEOUT || "5000", 10)
          })
    )
  )
)
```

### Pattern 2: Service with Dependencies

```typescript
interface Database {
  readonly query: <T>(sql: string) => Effect.Effect<T, DbError>
}

class Database extends Context.Tag("Database")<Database, Database>() {}

interface Logger {
  readonly log: (message: string) => Effect.Effect<void>
}

class Logger extends Context.Tag("Logger")<Logger, Logger>() {}

// Database depends on Config and Logger
const DatabaseLive = Layer.effect(
  Database,
  pipe(
    Effect.all([Config, Logger]),
    Effect.flatMap(([config, logger]) =>
      pipe(
        logger.log(`Connecting to database at ${config.apiUrl}`),
        Effect.flatMap(() =>
          Effect.tryPromise({
            try: () => createConnection(config.apiUrl),
            catch: (error) => ({ _tag: "ConnectionFailed" as const, error })
          })
        ),
        Effect.flatMap(connection =>
          pipe(
            logger.log("Database connected"),
            Effect.map(() => ({
              query: <T>(sql: string) =>
                pipe(
                  logger.log(`Executing query: ${sql}`),
                  Effect.flatMap(() =>
                    Effect.tryPromise({
                      try: () => connection.execute<T>(sql),
                      catch: (error) => ({ _tag: "QueryFailed" as const, error })
                    })
                  )
                )
            }))
          )
        )
      )
    )
  )
)
// Type: Layer<Database, ConnectionFailed, Config | Logger>
```

### Pattern 3: Service with State

```typescript
interface Cache {
  readonly get: <T>(key: string) => Effect.Effect<Option<T>, never>
  readonly set: <T>(key: string, value: T) => Effect.Effect<void>
  readonly clear: () => Effect.Effect<void>
}

class Cache extends Context.Tag("Cache")<Cache, Cache>() {}

const CacheLive = Layer.effect(
  Cache,
  pipe(
    Ref.make<Map<string, unknown>>(new Map()),
    Effect.map(store => ({
      get: <T>(key: string) =>
        pipe(
          Ref.get(store),
          Effect.map(map => Option.fromNullable(map.get(key) as T))
        ),

      set: <T>(key: string, value: T) =>
        Ref.update(store, map => new Map(map).set(key, value)),

      clear: () =>
        Ref.set(store, new Map())
    }))
  )
)
```

### Pattern 4: Service with Cleanup (Scoped)

```typescript
interface WebSocketConnection {
  readonly send: (message: string) => Effect.Effect<void>
  readonly receive: () => Effect.Effect<string>
  readonly close: () => Effect.Effect<void>
}

class WebSocketConnection extends Context.Tag("WebSocketConnection")<
  WebSocketConnection,
  WebSocketConnection
>() {}

const WebSocketConnectionLive = Layer.scoped(
  WebSocketConnection,
  pipe(
    Effect.all([Config, Logger]),
    Effect.flatMap(([config, logger]) =>
      pipe(
        logger.log("Opening WebSocket connection"),
        Effect.flatMap(() =>
          Effect.sync(() => new WebSocket(config.wsUrl))
        ),
        Effect.flatMap(ws =>
          pipe(
            Effect.async<void, never>((resume) => {
              ws.onopen = () => resume(Effect.void)
              ws.onerror = () => resume(Effect.void)
            }),
            Effect.flatMap(() =>
              logger.log("WebSocket connected")
            ),
            Effect.flatMap(() =>
              Effect.addFinalizer(() =>
                pipe(
                  logger.log("Closing WebSocket connection"),
                  Effect.flatMap(() =>
                    Effect.sync(() => ws.close())
                  )
                )
              )
            ),
            Effect.map(() => ({
              send: (message) =>
                Effect.sync(() => ws.send(message)),

              receive: () =>
                Effect.async<string>((resume) => {
                  ws.onmessage = (event) => resume(Effect.succeed(event.data))
                }),

              close: () =>
                Effect.sync(() => ws.close())
            }))
          )
        )
      )
    )
  )
)
```

---

[↑ Back to Top](#table-of-contents)

## Layer Composition

### Sequential Composition (provide)

When one layer depends on services from another:

```typescript
// ApiClient depends on Config
const ApiClientLive = Layer.effect(
  ApiClient,
  pipe(
    Config,
    Effect.map(config => makeApiClient(config))
  )
)
// Layer<ApiClient, never, Config>

// Provide Config to ApiClient
const ApiClientWithConfig = ApiClientLive.pipe(
  Layer.provide(ConfigLive)
)
// Layer<ApiClient, never, never>
```

### Parallel Composition (merge)

When layers don't depend on each other:

```typescript
// These can be built in parallel
const IndependentServices = Layer.merge(
  ConfigLive,
  LoggerLive
)
// Layer<Config | Logger, never, never>

// Or merge multiple
const AllIndependent = Layer.mergeAll(
  ConfigLive,
  LoggerLive,
  CacheLive
)
```

### Layered Architecture

Building a complete application layer graph:

```typescript
// Layer 1: Configuration (no dependencies)
const ConfigLayer = ConfigLive

// Layer 2: Infrastructure (depends on Config)
const InfrastructureLayer = Layer.mergeAll(
  LoggerLive,
  CacheLive
).pipe(
  Layer.provide(ConfigLayer)
)

// Layer 3: Data Access (depends on Infrastructure)
const DataLayer = Layer.mergeAll(
  DatabaseLive,
  ApiClientLive
).pipe(
  Layer.provide(InfrastructureLayer)
)

// Layer 4: Domain Services (depends on Data)
const DomainLayer = Layer.mergeAll(
  UserServiceLive,
  PostServiceLive,
  AuthServiceLive
).pipe(
  Layer.provide(DataLayer)
)

// Complete application layer
const AppLayer = DomainLayer

// Now use it
const program = pipe(
  UserService,
  Effect.flatMap(userService =>
    userService.getUser("123")
  )
)

Effect.runPromise(program.pipe(Effect.provide(AppLayer)))
```

### Layer Composition Patterns

```typescript
// Pattern 1: Diamond dependency
// UserService and PostService both depend on ApiClient
const Services = Layer.mergeAll(
  UserServiceLive,
  PostServiceLive
).pipe(
  Layer.provide(ApiClientLive) // ApiClient is shared, not duplicated
)

// Pattern 2: Conditional layers
const StorageLayer = isProduction
  ? RedisStorageLive
  : InMemoryStorageLive

// Pattern 3: Layer factory
const makeApiLayer = (baseUrl: string) =>
  Layer.effect(
    ApiClient,
    Effect.sync(() => makeApiClient({ baseUrl }))
  )

const DevApiLayer = makeApiLayer("http://localhost:3000")
const ProdApiLayer = makeApiLayer("https://api.example.com")

// Pattern 4: Extend existing layer
const ApiWithLoggingLive = Layer.effect(
  ApiClient,
  pipe(
    Effect.all([ApiClient, Logger]),
    Effect.map(([api, logger]) => ({
      get: <T>(url: string) =>
        pipe(
          logger.log(`GET ${url}`),
          Effect.flatMap(() => api.get<T>(url)),
          Effect.tap(result => logger.log(`GET ${url} succeeded`))
        ),
      post: <T, B>(url: string, body: B) =>
        pipe(
          logger.log(`POST ${url}`),
          Effect.flatMap(() => api.post<T, B>(url, body)),
          Effect.tap(result => logger.log(`POST ${url} succeeded`))
        )
    }))
  )
).pipe(
  Layer.provide(Layer.merge(ApiClientLive, LoggerLive))
)
```

---

[↑ Back to Top](#table-of-contents)

## Advanced Layer Patterns

### Pattern 1: Lazy Layer Construction

Defer layer construction until actually needed:

```typescript
// Create layer factory
const makeUserServiceLayer = (userId: string) =>
  Layer.effect(
    UserService,
    pipe(
      ApiClient,
      Effect.flatMap(api =>
        pipe(
          api.get(`/config/${userId}`),
          Effect.map(userConfig =>
            makeUserService(userConfig)
          )
        )
      )
    )
  )

// Use in application
const program = (userId: string) =>
  pipe(
    userWorkflow,
    Effect.provide(makeUserServiceLayer(userId))
  )
```

### Pattern 2: Layer with Initialization

Services that need async initialization:

```typescript
interface MessageQueue {
  readonly publish: (message: string) => Effect.Effect<void>
  readonly subscribe: () => Stream.Stream<string>
}

class MessageQueue extends Context.Tag("MessageQueue")<MessageQueue, MessageQueue>() {}

const MessageQueueLive = Layer.scoped(
  MessageQueue,
  pipe(
    Effect.all([Config, Logger]),
    Effect.flatMap(([config, logger]) =>
      pipe(
        logger.log("Connecting to message queue..."),
        Effect.flatMap(() =>
          Effect.tryPromise({
            try: () => connectToRabbitMQ(config.mqUrl),
            catch: (error) => ({ _tag: "ConnectionFailed" as const, error })
          })
        ),
        Effect.flatMap(connection =>
          pipe(
            Effect.tryPromise({
              try: () => connection.waitReady(),
              catch: (error) => ({ _tag: "InitFailed" as const, error })
            }),
            Effect.flatMap(() =>
              logger.log("Message queue ready")
            ),
            Effect.flatMap(() =>
              Effect.addFinalizer(() =>
                pipe(
                  logger.log("Disconnecting message queue"),
                  Effect.flatMap(() =>
                    Effect.sync(() => connection.close())
                  )
                )
              )
            ),
            Effect.map(() => ({
              publish: (message) =>
                Effect.tryPromise({
                  try: () => connection.publish(message),
                  catch: (error) => ({ _tag: "PublishFailed" as const, error })
                }),

              subscribe: () =>
                Stream.asyncEffect<string>((emit) =>
                  Effect.sync(() => {
                    connection.on("message", (msg: string) => {
                      emit(Effect.succeed(Chunk.of(msg)))
                    })
                  })
                )
            }))
          )
        )
      )
    )
  )
)
```

### Pattern 3: Multi-Tenant Layers

Different layer configurations per tenant:

```typescript
interface TenantContext {
  readonly tenantId: string
  readonly config: TenantConfig
}

class TenantContext extends Context.Tag("TenantContext")<TenantContext, TenantContext>() {}

// Create tenant-specific database layer
const makeTenantDatabaseLayer = (tenantId: string) =>
  Layer.effect(
    Database,
    pipe(
      TenantContext,
      Effect.flatMap(tenantCtx =>
        pipe(
          Effect.sync(() =>
            createConnection(tenantCtx.config.dbUrl)
          ),
          Effect.map(connection => ({
            query: <T>(sql: string) =>
              Effect.sync(() => connection.query<T>(sql))
          }))
        )
      )
    )
  ).pipe(
    Layer.provide(
      Layer.succeed(TenantContext, {
        tenantId,
        config: getTenantConfig(tenantId)
      })
    )
  )

// Use in multi-tenant application
const handleTenantRequest = (tenantId: string, request: Request) =>
  pipe(
    processRequest(request),
    Effect.provide(makeTenantDatabaseLayer(tenantId))
  )
```

### Pattern 4: Feature Flag Layers

Swap implementations based on feature flags:

```typescript
interface FeatureFlags {
  readonly isEnabled: (flag: string) => boolean
}

class FeatureFlags extends Context.Tag("FeatureFlags")<FeatureFlags, FeatureFlags>() {}

// Old implementation
const UserServiceV1Live = Layer.effect(
  UserService,
  Effect.sync(() => makeUserServiceV1())
)

// New implementation
const UserServiceV2Live = Layer.effect(
  UserService,
  Effect.sync(() => makeUserServiceV2())
)

// Dynamic layer selection
const UserServiceLive = Layer.effect(
  UserService,
  pipe(
    FeatureFlags,
    Effect.map(flags =>
      flags.isEnabled("user-service-v2")
        ? makeUserServiceV2()
        : makeUserServiceV1()
    )
  )
).pipe(
  Layer.provide(FeatureFlagsLive)
)
```

### Pattern 5: Pooled Resources

Layer for managing resource pools:

```typescript
interface ConnectionPool {
  readonly acquire: () => Effect.Effect<Connection, PoolError>
  readonly release: (conn: Connection) => Effect.Effect<void>
}

class ConnectionPool extends Context.Tag("ConnectionPool")<ConnectionPool, ConnectionPool>() {}

const ConnectionPoolLive = Layer.scoped(
  ConnectionPool,
  pipe(
    Effect.all([Config, Logger]),
    Effect.flatMap(([config, logger]) =>
      pipe(
        Effect.sync(() =>
          createPool({
            max: config.poolSize,
            create: () => createConnection(config.dbUrl)
          })
        ),
        Effect.flatMap(pool =>
          pipe(
            logger.log(`Connection pool created (size: ${config.poolSize})`),
            Effect.flatMap(() =>
              Effect.addFinalizer(() =>
                pipe(
                  logger.log("Draining connection pool"),
                  Effect.flatMap(() =>
                    Effect.sync(() => pool.drain())
                  )
                )
              )
            ),
            Effect.map(() => ({
              acquire: () =>
                Effect.tryPromise({
                  try: () => pool.acquire(),
                  catch: (error) => ({ _tag: "PoolExhausted" as const, error })
                }),

              release: (conn) =>
                Effect.sync(() => pool.release(conn))
            }))
          )
        )
      )
    )
  )
)

// Use with automatic release
const withConnection = <A, E>(
  use: (conn: Connection) => Effect.Effect<A, E>
): Effect.Effect<A, E | PoolError> =>
  pipe(
    ConnectionPool,
    Effect.flatMap(pool =>
      Effect.acquireUseRelease(
        pool.acquire(),
        use,
        (conn) => pool.release(conn)
      )
    )
  )
```

---

[↑ Back to Top](#table-of-contents)

## Managing Layer Lifecycles

### Scope and Lifetime

Layers can have different lifecycles:

```typescript
// 1. Application-wide (singleton)
const AppLayer = Layer.mergeAll(
  ConfigLive,
  DatabaseLive,
  CacheLive
)

// Everything in AppLayer is created once and shared

// 2. Request-scoped
const RequestLayer = Layer.scoped(
  RequestContext,
  pipe(
    Effect.sync(() => generateId()),
    Effect.flatMap(requestId =>
      pipe(
        Effect.addFinalizer(() =>
          Console.log(`Request ${requestId} completed`)
        ),
        Effect.map(() => ({ requestId }))
      )
    )
  )
)

// New RequestContext for each request

// 3. Manual scope management
const program = Effect.scoped(
  pipe(
    Database,
    Effect.flatMap(db =>
      db.query("SELECT * FROM users")
    )
  )
)
// Scope closed, resources cleaned up
```

### Layer Memoization

Layers are memoized by default - same layer instance is reused:

```typescript
const ExpensiveServiceLive = Layer.effect(
  ExpensiveService,
  pipe(
    Effect.sync(() => console.log("Building ExpensiveService")),
    Effect.flatMap(() => Effect.sleep("5 seconds")),
    Effect.map(() => makeExpensiveService())
  )
)

const program = pipe(
  Effect.all([ExpensiveService, ExpensiveService]),
  Effect.map(([svc1, svc2]) => {
    // Both use the same instance, built only once
  })
)

Effect.runPromise(program.pipe(Effect.provide(ExpensiveServiceLive)))
```

### Fresh Layers

Force a new instance each time:

```typescript
const FreshServiceLive = Layer.effect(
  Service,
  pipe(
    Effect.sync(() => console.log("Building Service")),
    Effect.map(() => makeService())
  )
).pipe(Layer.fresh)

// Or use Layer.unwrapEffect for dynamic construction
const DynamicLayer = Layer.unwrapEffect(
  pipe(
    Effect.sync(() => loadConfig()),
    Effect.map(config =>
      Layer.succeed(Service, makeService(config))
    )
  )
)
```

---

[↑ Back to Top](#table-of-contents)

## Testing Strategies

### Test Layers

Create dedicated test layers:

```typescript
// Test implementation of ApiClient
const TestApiClient: ApiClient = {
  get: <T>(url: string) => {
    const testData: Record<string, unknown> = {
      "/users/123": { id: "123", name: "Test User" },
      "/posts": [{ id: "1", title: "Test Post" }]
    }
    
    return Effect.succeed(testData[url] as T)
  },
  
  post: <T, B>(url: string, body: B) =>
    Effect.succeed({ success: true } as T)
}

const TestApiClientLayer = Layer.succeed(ApiClient, TestApiClient)

// Test layer with all mocks
const TestLayer = Layer.mergeAll(
  TestApiClientLayer,
  MockLoggerLayer,
  InMemoryCacheLayer
)

// Use in tests
describe("UserService", () => {
  it("should fetch user", async () => {
    const program = pipe(
      UserService,
      Effect.flatMap(userService =>
        userService.getUser("123")
      )
    )

    const result = await Effect.runPromise(
      program.pipe(Effect.provide(TestLayer))
    )

    expect(result.name).toBe("Test User")
  })
})
```

### Spy Layers

Track calls to services:

```typescript
const makeSpyLogger = pipe(
  Ref.make<string[]>([]),
  Effect.map(calls => ({
    service: {
      log: (message: string) =>
        Ref.update(calls, prev => [...prev, message])
    },
    getCalls: () => Ref.get(calls)
  }))
)

const SpyLoggerLayer = Layer.unwrapEffect(
  pipe(
    makeSpyLogger,
    Effect.map(spy =>
      Layer.succeed(Logger, spy.service)
    )
  )
)

// Test with spy
const test = pipe(
  Logger,
  Effect.flatMap(logger =>
    pipe(
      logger.log("message 1"),
      Effect.flatMap(() => logger.log("message 2"))
    )
  )
)
// How to access spy? Need to restructure...

// Better pattern: export spy reference
interface LoggerSpy {
  readonly getCalls: () => Effect.Effect<string[]>
}

class LoggerSpy extends Context.Tag("LoggerSpy")<LoggerSpy, LoggerSpy>() {}

const SpyLoggerLayerWithRef = Layer.effect(
  Logger,
  pipe(
    Ref.make<string[]>([]),
    Effect.map(calls => ({
      logger: {
        log: (message: string) =>
          Ref.update(calls, prev => [...prev, message])
      },
      spy: {
        getCalls: () => Ref.get(calls)
      }
    }))
  )
)
```

### Partial Mocking

Mock only specific services:

```typescript
// Use real implementation for some services
const PartialTestLayer = Layer.mergeAll(
  TestApiClientLayer,  // Mocked
  LoggerLive,          // Real
  TestDatabaseLayer    // Mocked
)

const program = pipe(
  myWorkflow,
  Effect.provide(PartialTestLayer)
)
```

---

[↑ Back to Top](#table-of-contents)

## Real-World Architecture

### Layered Application Architecture

```typescript
// 1. Infrastructure Layer (lowest level)
const InfrastructureLayer = Layer.mergeAll(
  ConfigLive,
  LoggerLive,
  MetricsLive
)

// 2. Platform Layer (databases, external services)
const PlatformLayer = Layer.mergeAll(
  DatabaseLive,
  CacheLive,
  MessageQueueLive,
  ApiClientLive
).pipe(
  Layer.provide(InfrastructureLayer)
)

// 3. Repository Layer (data access)
const RepositoryLayer = Layer.mergeAll(
  UserRepositoryLive,
  PostRepositoryLive,
  CommentRepositoryLive
).pipe(
  Layer.provide(PlatformLayer)
)

// 4. Service Layer (business logic)
const ServiceLayer = Layer.mergeAll(
  UserServiceLive,
  PostServiceLive,
  AuthServiceLive,
  NotificationServiceLive
).pipe(
  Layer.provide(RepositoryLayer)
)

// 5. Application Layer (orchestration)
const ApplicationLayer = Layer.mergeAll(
  CreatePostUseCaseLive,
  PublishPostUseCaseLive,
  UserRegistrationUseCaseLive
).pipe(
  Layer.provide(ServiceLayer)
)

// Complete application
export const AppLayer = ApplicationLayer
```

### Environment-Specific Layers

```typescript
// environments/development.ts
export const DevelopmentLayer = Layer.mergeAll(
  Layer.succeed(Config, {
    apiUrl: "http://localhost:3000",
    logLevel: "debug",
    cacheEnabled: false
  }),
  ConsoleLoggerLive,
  InMemoryCacheLive,
  LocalDatabaseLive
)

// environments/production.ts
export const ProductionLayer = Layer.mergeAll(
  Layer.succeed(Config, {
    apiUrl: process.env.API_URL!,
    logLevel: "info",
    cacheEnabled: true
  }),
  CloudWatchLoggerLive,
  RedisCacheLive,
  RDSDatabaseLive
)

// main.ts
const AppLayer = (
  process.env.NODE_ENV === "production"
    ? ProductionLayer
    : DevelopmentLayer
).pipe(
  Layer.provide(ApplicationServicesLayer)
)
```

### Module-Based Organization

```typescript
// modules/users/layer.ts
export const UsersModuleLayer = Layer.mergeAll(
  UserRepositoryLive,
  UserServiceLive,
  UserControllerLive
)

// modules/posts/layer.ts
export const PostsModuleLayer = Layer.mergeAll(
  PostRepositoryLive,
  PostServiceLive,
  PostControllerLive
)

// app.ts
export const AppLayer = Layer.mergeAll(
  UsersModuleLayer,
  PostsModuleLayer,
  AuthModuleLayer
).pipe(
  Layer.provide(SharedInfrastructureLayer)
)
```

---

[↑ Back to Top](#table-of-contents)

## Performance Considerations

### Layer Initialization Performance

```typescript
// Parallel initialization
const FastLayer = Layer.mergeAll(
  ServiceALive,  // These initialize
  ServiceBLive,  // in parallel
  ServiceCLive   // if independent
)

// Sequential initialization (if needed)
const SlowLayer = ServiceCLive.pipe(
  Layer.provide(ServiceBLive),
  Layer.provide(ServiceALive)
)
// A → B → C (sequential)
```

### Lazy Services

```typescript
// Don't initialize until actually used
const LazyDatabaseLive = Layer.effect(
  Database,
  pipe(
    Effect.sync(() => createConnection()),
    Effect.map(connection => makeDatabase(connection))
  )
)
```

### Resource Pooling

```typescript
// Share expensive resources
const ConnectionPoolLayer = Layer.scoped(
  ConnectionPool,
  pipe(
    Effect.sync(() => createPool({ size: 10 })),
    Effect.flatMap(pool =>
      pipe(
        Effect.addFinalizer(() =>
          Effect.sync(() => pool.drain())
        ),
        Effect.map(() => pool)
      )
    )
  )
)

// Multiple services use the same pool
const UserRepoLive = Layer.effect(
  UserRepository,
  pipe(
    ConnectionPool,
    Effect.map(pool => makeUserRepo(pool))
  )
)

const PostRepoLive = Layer.effect(
  PostRepository,
  pipe(
    ConnectionPool,
    Effect.map(pool => makePostRepo(pool))
  )
)

const AppLayer = Layer.mergeAll(
  UserRepoLive,
  PostRepoLive
).pipe(
  Layer.provide(ConnectionPoolLayer) // Shared pool
)
```

---

[↑ Back to Top](#table-of-contents)

## Best Practices Summary

1. **Keep layers pure** - No side effects in layer construction
2. **One service per layer** - Don't combine unrelated services
3. **Use scoped for cleanup** - Resources that need cleanup should use `Layer.scoped`
4. **Compose vertically** - Build layers in levels (infrastructure → data → domain)
5. **Test with mocks** - Create test layers for all services
6. **Document dependencies** - Make service requirements explicit
7. **Environment-specific** - Use different layers for dev/test/prod
8. **Avoid circular deps** - Design services to avoid circular dependencies
9. **Cache expensive services** - Layers are memoized by default, use it
10. **Scope appropriately** - Application-wide vs request-scoped

---

[↑ Back to Top](#table-of-contents)

## Additional Resources

- **Effect Documentation**: https://effect.website/docs/context-management/layers
- **Effect Discord**: https://discord.gg/effect-ts
- **Example Projects**: https://github.com/Effect-TS/effect/tree/main/packages/examples

---

This deep dive covers advanced patterns for using Effect Layers to build scalable, maintainable, and testable applications with proper dependency injection!
