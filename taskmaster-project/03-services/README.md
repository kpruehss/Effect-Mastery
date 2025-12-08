# Chapter 3: Service Layer

**Building Effect-based services for data management**

In this chapter, we'll build the service layer that handles all data operations, API communication, and state management using Effect patterns.

---

## Table of Contents

1. [Service Architecture](#service-architecture)
2. [API Client Service](#api-client-service)
3. [Task Service](#task-service)
4. [Project Service](#project-service)
5. [WebSocket Service](#websocket-service)
6. [Testing Services](#testing-services)
7. [Next Steps](#next-steps)

---

## Service Architecture

### What is a Service?

In Effect-TS, a service is:
- A collection of related operations
- Managed through Context for dependency injection
- Composable with other services
- Testable with mock implementations

### Service Pattern

```typescript
// 1. Define interface
interface MyService {
  readonly operation: () => Effect.Effect<Result, Error>
}

// 2. Create Context tag
class MyService extends Context.Tag("MyService")<
  MyService,
  MyService
>() {}

// 3. Create Layer
const MyServiceLive = Layer.effect(
  MyService,
  Effect.gen(function* () {
    // Setup and dependencies
    return {
      operation: () => Effect.succeed(result)
    }
  })
)
```

### Services We'll Build

1. **ApiClient**: HTTP request wrapper
2. **TaskService**: Task CRUD with optimistic updates
3. **ProjectService**: Project management
4. **WebSocketService**: Real-time updates

Each service builds on Effect's composability.

---

## API Client Service

The foundation for all HTTP communication.

### Why ApiClient?

- Centralized error handling
- Consistent timeout behavior
- Credential management
- Request/response transformation

### Implementation

Create `src/lib/services/ApiClient.ts`:

See [ApiClient.md](./ApiClient.md) for complete implementation.

**Key features:**
- Generic `get`, `post`, `put`, `delete` methods
- Automatic timeout handling
- Credential inclusion
- Typed error responses

---

## Task Service

Manages all task-related operations with optimistic UI updates.

### Features

- List tasks (with optional project filter)
- Get single task
- Create task (optimistic)
- Update task
- Delete task (optimistic with rollback)

### Optimistic Updates

Optimistic updates improve perceived performance:

1. **Immediately** update UI with temporary data
2. **Send** request to server
3. **Replace** temporary data with server response
4. **Rollback** if request fails

### Implementation

Create `src/lib/services/TaskService.ts`:

See [TaskService.md](./TaskService.md) for complete implementation.

**Key patterns:**
- Ref for local state management
- Optimistic updates with rollback
- Error transformation
- Concurrent operations

---

## Project Service

Manages project operations.

### Features

- List all projects
- Get single project
- Create project
- Delete project

### Implementation

Create `src/lib/services/ProjectService.ts`:

See [ProjectService.md](./ProjectService.md) for complete implementation.

**Simpler than TaskService:**
- No optimistic updates needed
- Straightforward CRUD operations
- Shared Ref for local state

---

## WebSocket Service

Real-time updates for collaborative features.

### Features

- WebSocket connection management
- Message streaming
- Send messages
- Automatic cleanup on unmount

### Implementation

Create `src/lib/services/WebSocketService.ts`:

See [WebSocketService.md](./WebSocketService.md) for complete implementation.

**Key patterns:**
- Scoped layer for cleanup
- Stream-based message handling
- Queue for buffering
- Connection lifecycle management

---

## Testing Services

### Mock Services

Create test implementations:

```typescript
// src/lib/services/ApiClient.test.ts
import { Layer } from "effect"
import { ApiClient } from "./ApiClient"

export const TestApiClient = Layer.succeed(
  ApiClient,
  {
    get: <T>(_url: string) =>
      Effect.succeed({ id: "123", name: "Test" } as T),

    post: <T, B>(_url: string, _body: B) =>
      Effect.succeed({ success: true } as T),

    put: <T, B>(_url: string, _body: B) =>
      Effect.succeed({ success: true } as T),

    delete: (_url: string) =>
      Effect.void
  }
)
```

### Service Tests

```typescript
// src/lib/services/TaskService.test.ts
import { describe, it, expect } from 'vitest'
import { Effect, Layer } from 'effect'
import { TaskService, TaskServiceLive } from './TaskService'
import { TestApiClient } from './ApiClient.test'

describe('TaskService', () => {
  const TestLayer = TaskServiceLive.pipe(
    Layer.provide(TestApiClient)
  )

  it('creates task with optimistic update', async () => {
    const program = Effect.gen(function* () {
      const service = yield* TaskService
      const task = yield* service.createTask({
        title: 'New Task',
        priority: 'high',
        projectId: 'project-123'
      })

      return task
    })

    const result = await Effect.runPromise(
      program.pipe(Effect.provide(TestLayer))
    )

    expect(result.title).toBe('New Task')
  })
})
```

---

## Service Patterns & Best Practices

### 1. Keep Services Focused

Each service should have a single responsibility:

```typescript
// ✅ Good - Focused
interface TaskService {
  readonly listTasks: () => Effect.Effect<Task[]>
  readonly createTask: (input: CreateTaskInput) => Effect.Effect<Task>
}

// ❌ Bad - Too many responsibilities
interface MegaService {
  readonly listTasks: () => Effect.Effect<Task[]>
  readonly sendEmail: (email: Email) => Effect.Effect<void>
  readonly processPayment: (payment: Payment) => Effect.Effect<void>
}
```

### 2. Use Refs for State

Refs provide reactive state management:

```typescript
const makeService = Effect.gen(function* () {
  const items = yield* Ref.make<Item[]>([])

  return {
    getItems: () => Ref.get(items),
    addItem: (item: Item) => Ref.update(items, prev => [...prev, item])
  }
})
```

### 3. Type Errors Properly

Use discriminated unions for errors:

```typescript
type ServiceError =
  | { _tag: "NotFound"; id: string }
  | { _tag: "ValidationError"; issues: unknown }
  | { _tag: "NetworkError" }
```

### 4. Handle Cleanup

Use `Layer.scoped` for resources needing cleanup:

```typescript
const makeWebSocket = Effect.gen(function* () {
  const ws = yield* Effect.sync(() => new WebSocket(url))

  // Register cleanup
  yield* Effect.addFinalizer(() =>
    Effect.sync(() => ws.close())
  )

  return ws
})

const WebSocketServiceLive = Layer.scoped(
  WebSocketService,
  makeWebSocket
)
```

### 5. Compose Services

Services can depend on other services:

```typescript
const TaskServiceLive = Layer.effect(
  TaskService,
  Effect.gen(function* () {
    const api = yield* ApiClient
    const logger = yield* Logger

    return {
      createTask: (input) => pipe(
        logger.log("Creating task"),
        Effect.flatMap(() => api.post("/tasks", input))
      )
    }
  })
)
```

---

## Common Patterns

### Pattern 1: Retry Logic

```typescript
const fetchWithRetry = pipe(
  apiClient.get("/data"),
  Effect.retry(Schedule.exponential("100 millis"))
)
```

### Pattern 2: Timeout

```typescript
const fetchWithTimeout = pipe(
  apiClient.get("/data"),
  Effect.timeout("5 seconds")
)
```

### Pattern 3: Concurrent Requests

```typescript
const [users, posts] = yield* Effect.all([
  apiClient.get("/users"),
  apiClient.get("/posts")
], { concurrency: "unbounded" })
```

### Pattern 4: Sequential with Dependencies

```typescript
const workflow = Effect.gen(function* () {
  const user = yield* getUser(userId)
  const posts = yield* getUserPosts(user.id)
  const comments = yield* getPostComments(posts[0].id)
  return { user, posts, comments }
})
```

---

## Service Layer Checklist

Before moving to Chapter 4, verify:

- ✅ ApiClient service implemented
- ✅ TaskService with optimistic updates
- ✅ ProjectService implemented
- ✅ WebSocketService for real-time updates
- ✅ Test utilities created
- ✅ Error handling consistent
- ✅ Services follow Effect patterns

---

## What We Accomplished

In this chapter, we:

1. ✅ Built ApiClient for HTTP communication
2. ✅ Created TaskService with optimistic updates
3. ✅ Implemented ProjectService
4. ✅ Added WebSocketService for real-time features
5. ✅ Learned service composition patterns
6. ✅ Implemented proper error handling

---

## Additional Resources

- [ApiClient Implementation](./ApiClient.md)
- [TaskService Implementation](./TaskService.md)
- [ProjectService Implementation](./ProjectService.md)
- [WebSocketService Implementation](./WebSocketService.md)

---

## Next Steps

With services implemented, we'll move to [Chapter 4: Effect Layers & Runtime](../04-layers/README.md) where we'll:

- Compose services into layers
- Create infrastructure layer
- Build application layer
- Set up runtime configuration
- Handle environment-specific layers

[Continue to Chapter 4 →](../04-layers/README.md)

---

[← Chapter 2](../02-domain/README.md) | [Main](../README.md) | [Chapter 4 →](../04-layers/README.md)
