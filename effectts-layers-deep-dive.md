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

const Logger = Context.GenericTag<Logger>("Logger")

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
  Effect.gen(function* () {
    // Perform setup
    yield* Effect.sync(() => console.log("Logger initialized"))
    
    return {
      log: (message) => Console.log(`[LOG] ${message}`)
    }
  })
)
// Type: Layer<Logger, never, never>

// 3. Layer.scoped - For services that need cleanup
const LoggerWithCleanup = Layer.scoped(
  Logger,
  Effect.gen(function* () {
    // Setup
    const logFile = yield* Effect.sync(() => openLogFile())
    
    // Register cleanup
    yield* Effect.addFinalizer(() => 
      Effect.sync(() => logFile.close())
    )
    
    return {
      log: (message) => 
        Effect.sync(() => logFile.write(message))
    }
  })
)
// Type: Layer<Logger, never, Scope>
```

---

## Layer Construction Patterns

### Pattern 1: Simple Service (No Dependencies)

```typescript
interface Config {
  readonly apiUrl: string
  readonly timeout: number
}

const Config = Context.GenericTag<Config>("Config")

// Environment-based config
const makeConfig = (): Config => ({
  apiUrl: process.env.API_URL || "http://localhost:3000",
  timeout: parseInt(process.env.TIMEOUT || "5000", 10)
})

const ConfigLive = Layer.succeed(Config, makeConfig())

// Or with validation
const ConfigLiveValidated = Layer.effect(
  Config,
  Effect.gen(function* () {
    const apiUrl = process.env.API_URL
    
    if (!apiUrl) {
      return yield* Effect.fail({ _tag: "MissingConfig" as const, key: "API_URL" })
    }
    
    return {
      apiUrl,
      timeout: parseInt(process.env.TIMEOUT || "5000", 10)
    }
  })
)
```

### Pattern 2: Service with Dependencies

```typescript
interface Database {
  readonly query: <T>(sql: string) => Effect.Effect<T, DbError>
}

const Database = Context.GenericTag<Database>("Database")

interface Logger {
  readonly log: (message: string) => Effect.Effect<void>
}

const Logger = Context.GenericTag<Logger>("Logger")

// Database depends on Config and Logger
const makeDatabase = Effect.gen(function* () {
  const config = yield* Config
  const logger = yield* Logger
  
  // Setup connection
  yield* logger.log(`Connecting to database at ${config.apiUrl}`)
  const connection = yield* Effect.tryPromise({
    try: () => createConnection(config.apiUrl),
    catch: (error) => ({ _tag: "ConnectionFailed" as const, error })
  })
  
  yield* logger.log("Database connected")
  
  return Database.of({
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
  })
})

const DatabaseLive = Layer.effect(Database, makeDatabase)
// Type: Layer<Database, ConnectionFailed, Config | Logger>
```

### Pattern 3: Service with State

```typescript
interface Cache {
  readonly get: <T>(key: string) => Effect.Effect<Option<T>, never>
  readonly set: <T>(key: string, value: T) => Effect.Effect<void>
  readonly clear: () => Effect.Effect<void>
}

const Cache = Context.GenericTag<Cache>("Cache")

const makeCache = Effect.gen(function* () {
  // Create mutable state
  const store = yield* Ref.make<Map<string, unknown>>(new Map())
  
  return Cache.of({
    get: <T>(key: string) =>
      pipe(
        Ref.get(store),
        Effect.map(map => Option.fromNullable(map.get(key) as T))
      ),
    
    set: <T>(key: string, value: T) =>
      Ref.update(store, map => new Map(map).set(key, value)),
    
    clear: () =>
      Ref.set(store, new Map())
  })
})

const CacheLive = Layer.effect(Cache, makeCache)
```

### Pattern 4: Service with Cleanup (Scoped)

```typescript
interface WebSocketConnection {
  readonly send: (message: string) => Effect.Effect<void>
  readonly receive: () => Effect.Effect<string>
  readonly close: () => Effect.Effect<void>
}

const WebSocketConnection = Context.GenericTag<WebSocketConnection>(
  "WebSocketConnection"
)

const makeWebSocket = Effect.gen(function* () {
  const config = yield* Config
  const logger = yield* Logger
  
  // Acquire resource
  yield* logger.log("Opening WebSocket connection")
  const ws = yield* Effect.sync(() => new WebSocket(config.wsUrl))
  
  // Wait for connection
  yield* Effect.async<void, never>((resume) => {
    ws.onopen = () => resume(Effect.void)
    ws.onerror = () => resume(Effect.void)
  })
  
  yield* logger.log("WebSocket connected")
  
  // Register cleanup (runs automatically when scope closes)
  yield* Effect.addFinalizer(() =>
    Effect.gen(function* () {
      yield* logger.log("Closing WebSocket connection")
      yield* Effect.sync(() => ws.close())
    })
  )
  
  return WebSocketConnection.of({
    send: (message) =>
      Effect.sync(() => ws.send(message)),
    
    receive: () =>
      Effect.async<string>((resume) => {
        ws.onmessage = (event) => resume(Effect.succeed(event.data))
      }),
    
    close: () =>
      Effect.sync(() => ws.close())
  })
})

const WebSocketConnectionLive = Layer.scoped(
  WebSocketConnection,
  makeWebSocket
)
```

---

## Layer Composition

### Sequential Composition (provide)

When one layer depends on services from another:

```typescript
// ApiClient depends on Config
const ApiClientLive = Layer.effect(
  ApiClient,
  Effect.gen(function* () {
    const config = yield* Config
    return makeApiClient(config)
  })
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
const program = Effect.gen(function* () {
  const userService = yield* UserService
  const user = yield* userService.getUser("123")
  return user
})

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
  Effect.gen(function* () {
    const api = yield* ApiClient // Get existing implementation
    const logger = yield* Logger
    
    // Wrap with logging
    return {
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
    }
  })
).pipe(
  Layer.provide(Layer.merge(ApiClientLive, LoggerLive))
)
```

---

## Advanced Layer Patterns

### Pattern 1: Lazy Layer Construction

Defer layer construction until actually needed:

```typescript
// Create layer factory
const makeUserServiceLayer = (userId: string) =>
  Layer.effect(
    UserService,
    Effect.gen(function* () {
      const api = yield* ApiClient
      
      // Fetch user-specific config
      const userConfig = yield* api.get(`/config/${userId}`)
      
      return makeUserService(userConfig)
    })
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

const MessageQueue = Context.GenericTag<MessageQueue>("MessageQueue")

const makeMessageQueue = Effect.gen(function* () {
  const config = yield* Config
  const logger = yield* Logger
  
  // Connect to message broker
  yield* logger.log("Connecting to message queue...")
  const connection = yield* Effect.tryPromise({
    try: () => connectToRabbitMQ(config.mqUrl),
    catch: (error) => ({ _tag: "ConnectionFailed" as const, error })
  })
  
  // Wait for ready
  yield* Effect.tryPromise({
    try: () => connection.waitReady(),
    catch: (error) => ({ _tag: "InitFailed" as const, error })
  })
  
  yield* logger.log("Message queue ready")
  
  // Register cleanup
  yield* Effect.addFinalizer(() =>
    Effect.gen(function* () {
      yield* logger.log("Disconnecting message queue")
      yield* Effect.sync(() => connection.close())
    })
  )
  
  return MessageQueue.of({
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
  })
})

const MessageQueueLive = Layer.scoped(MessageQueue, makeMessageQueue)
```

### Pattern 3: Multi-Tenant Layers

Different layer configurations per tenant:

```typescript
interface TenantContext {
  readonly tenantId: string
  readonly config: TenantConfig
}

const TenantContext = Context.GenericTag<TenantContext>("TenantContext")

// Create tenant-specific database layer
const makeTenantDatabaseLayer = (tenantId: string) =>
  Layer.effect(
    Database,
    Effect.gen(function* () {
      const tenantCtx = yield* TenantContext
      
      // Connect to tenant-specific database
      const connection = yield* Effect.sync(() =>
        createConnection(tenantCtx.config.dbUrl)
      )
      
      return Database.of({
        query: <T>(sql: string) =>
          Effect.sync(() => connection.query<T>(sql))
      })
    })
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

const FeatureFlags = Context.GenericTag<FeatureFlags>("FeatureFlags")

// Old implementation
const UserServiceV1Live = Layer.effect(
  UserService,
  Effect.gen(function* () {
    return makeUserServiceV1()
  })
)

// New implementation
const UserServiceV2Live = Layer.effect(
  UserService,
  Effect.gen(function* () {
    return makeUserServiceV2()
  })
)

// Dynamic layer selection
const UserServiceLive = Layer.effect(
  UserService,
  Effect.gen(function* () {
    const flags = yield* FeatureFlags
    
    if (flags.isEnabled("user-service-v2")) {
      return makeUserServiceV2()
    } else {
      return makeUserServiceV1()
    }
  })
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

const ConnectionPool = Context.GenericTag<ConnectionPool>("ConnectionPool")

const makeConnectionPool = Effect.gen(function* () {
  const config = yield* Config
  const logger = yield* Logger
  
  // Create pool
  const pool = yield* Effect.sync(() =>
    createPool({
      max: config.poolSize,
      create: () => createConnection(config.dbUrl)
    })
  )
  
  yield* logger.log(`Connection pool created (size: ${config.poolSize})`)
  
  // Register cleanup
  yield* Effect.addFinalizer(() =>
    Effect.gen(function* () {
      yield* logger.log("Draining connection pool")
      yield* Effect.sync(() => pool.drain())
    })
  )
  
  return ConnectionPool.of({
    acquire: () =>
      Effect.tryPromise({
        try: () => pool.acquire(),
        catch: (error) => ({ _tag: "PoolExhausted" as const, error })
      }),
    
    release: (conn) =>
      Effect.sync(() => pool.release(conn))
  })
})

const ConnectionPoolLive = Layer.scoped(ConnectionPool, makeConnectionPool)

// Use with automatic release
const withConnection = <A, E>(
  use: (conn: Connection) => Effect.Effect<A, E>
): Effect.Effect<A, E | PoolError> =>
  Effect.gen(function* () {
    const pool = yield* ConnectionPool
    
    return yield* Effect.acquireUseRelease(
      pool.acquire(),
      use,
      (conn) => pool.release(conn)
    )
  })
```

---

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
  Effect.gen(function* () {
    const requestId = yield* Effect.sync(() => generateId())
    
    yield* Effect.addFinalizer(() =>
      Console.log(`Request ${requestId} completed`)
    )
    
    return { requestId }
  })
)

// New RequestContext for each request

// 3. Manual scope management
const program = Effect.gen(function* () {
  // Create a scope
  yield* Effect.scoped(
    Effect.gen(function* () {
      // Resources in this scope
      const db = yield* Database
      const result = yield* db.query("SELECT * FROM users")
      return result
    })
  )
  // Scope closed, resources cleaned up
})
```

### Layer Memoization

Layers are memoized by default - same layer instance is reused:

```typescript
const ExpensiveServiceLive = Layer.effect(
  ExpensiveService,
  Effect.gen(function* () {
    console.log("Building ExpensiveService") // Only runs once
    yield* Effect.sleep("5 seconds")
    return makeExpensiveService()
  })
)

const program = Effect.gen(function* () {
  const svc1 = yield* ExpensiveService
  const svc2 = yield* ExpensiveService
  // Both use the same instance, built only once
})

Effect.runPromise(program.pipe(Effect.provide(ExpensiveServiceLive)))
```

### Fresh Layers

Force a new instance each time:

```typescript
const FreshServiceLive = Layer.effect(
  Service,
  Effect.gen(function* () {
    console.log("Building Service") // Runs every time
    return makeService()
  })
).pipe(Layer.fresh)

// Or use Layer.unwrapEffect for dynamic construction
const DynamicLayer = Layer.unwrapEffect(
  Effect.gen(function* () {
    const config = yield* Effect.sync(() => loadConfig())
    return Layer.succeed(Service, makeService(config))
  })
)
```

---

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
    const program = Effect.gen(function* () {
      const userService = yield* UserService
      return yield* userService.getUser("123")
    })
    
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
const makeSpyLogger = Effect.gen(function* () {
  const calls = yield* Ref.make<string[]>([])
  
  return {
    service: {
      log: (message: string) =>
        Ref.update(calls, prev => [...prev, message])
    },
    getCalls: () => Ref.get(calls)
  }
})

const SpyLoggerLayer = Layer.unwrapEffect(
  Effect.gen(function* () {
    const spy = yield* makeSpyLogger
    return Layer.succeed(Logger, spy.service)
  })
)

// Test with spy
const test = Effect.gen(function* () {
  const logger = yield* Logger
  yield* logger.log("message 1")
  yield* logger.log("message 2")
  
  // How to access spy? Need to restructure...
})

// Better pattern: export spy reference
interface LoggerSpy {
  readonly getCalls: () => Effect.Effect<string[]>
}

const LoggerSpy = Context.GenericTag<LoggerSpy>("LoggerSpy")

const SpyLoggerLayerWithRef = Layer.effect(
  Logger,
  Effect.gen(function* () {
    const calls = yield* Ref.make<string[]>([])
    
    // Provide both Logger and LoggerSpy
    return {
      logger: {
        log: (message: string) =>
          Ref.update(calls, prev => [...prev, message])
      },
      spy: {
        getCalls: () => Ref.get(calls)
      }
    }
  })
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
  Effect.gen(function* () {
    // This only runs when Database is first accessed
    const connection = yield* Effect.sync(() => createConnection())
    return makeDatabase(connection)
  })
)
```

### Resource Pooling

```typescript
// Share expensive resources
const ConnectionPoolLayer = Layer.scoped(
  ConnectionPool,
  Effect.gen(function* () {
    const pool = yield* Effect.sync(() => createPool({ size: 10 }))
    
    yield* Effect.addFinalizer(() =>
      Effect.sync(() => pool.drain())
    )
    
    return pool
  })
)

// Multiple services use the same pool
const UserRepoLive = Layer.effect(
  UserRepository,
  Effect.gen(function* () {
    const pool = yield* ConnectionPool
    return makeUserRepo(pool)
  })
)

const PostRepoLive = Layer.effect(
  PostRepository,
  Effect.gen(function* () {
    const pool = yield* ConnectionPool
    return makePostRepo(pool)
  })
)

const AppLayer = Layer.mergeAll(
  UserRepoLive,
  PostRepoLive
).pipe(
  Layer.provide(ConnectionPoolLayer) // Shared pool
)
```

---

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

## Additional Resources

- **Effect Documentation**: https://effect.website/docs/context-management/layers
- **Effect Discord**: https://discord.gg/effect-ts
- **Example Projects**: https://github.com/Effect-TS/effect/tree/main/packages/examples

---

This deep dive covers advanced patterns for using Effect Layers to build scalable, maintainable, and testable applications with proper dependency injection!
