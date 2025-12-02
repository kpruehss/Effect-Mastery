# Chapter 4: Effect Layers & Runtime

**Composing services and creating the application runtime**

In this chapter, we'll compose our services into layers and create a runtime that makes them available throughout the application.

---

## Table of Contents

1. [Understanding Layers](#understanding-layers)
2. [Layer Composition](#layer-composition)
3. [Application Layer](#application-layer)
4. [Runtime Configuration](#runtime-configuration)
5. [Using the Runtime](#using-the-runtime)
6. [Next Steps](#next-steps)

---

## Understanding Layers

### What is a Layer?

A Layer is a blueprint for constructing services:

```typescript
type Layer<Out, Error, In> = {
  // Produces services of type Out
  // May fail with error type E
  // Requires services of type In
}
```

### Why Layers?

Layers provide:
- **Dependency injection**: Services declare what they need
- **Composition**: Build complex graphs from simple pieces
- **Reusability**: Swap implementations (dev/test/prod)
- **Testability**: Easy mocking

---

## Layer Composition

### Infrastructure Layer

The lowest level - services with no dependencies.

Create `src/lib/effects/layers.ts`:

```typescript
import { Layer } from "effect"
import { ApiClientLive } from "$services/ApiClient"
import { WebSocketServiceLive } from "$services/WebSocketService"

// Infrastructure: no dependencies
const InfrastructureLayer = Layer.merge(
  ApiClientLive,
  WebSocketServiceLive
)
```

### Service Layer

Services that depend on infrastructure.

```typescript
import { TaskServiceLive } from "$services/TaskService"
import { ProjectServiceLive } from "$services/ProjectService"

// Services: depend on infrastructure
const ServiceLayer = Layer.mergeAll(
  TaskServiceLive,
  ProjectServiceLive
).pipe(
  Layer.provide(InfrastructureLayer)
)
```

### Application Layer

The complete app with all services.

```typescript
// Complete application layer
export const AppLayer = ServiceLayer
```

### Layer Diagram

```
┌─────────────────────────────────────┐
│         Application Layer           │
│                                     │
│  ┌──────────────────────────────┐  │
│  │      Service Layer           │  │
│  │                              │  │
│  │  TaskService                 │  │
│  │  ProjectService              │  │
│  │                              │  │
│  └──────────────────────────────┘  │
│              ↓ depends on          │
│  ┌──────────────────────────────┐  │
│  │   Infrastructure Layer       │  │
│  │                              │  │
│  │  ApiClient                   │  │
│  │  WebSocketService            │  │
│  │                              │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

---

## Application Layer

### Complete layers.ts

```typescript
import { Layer } from "effect"
import { ApiClientLive } from "$services/ApiClient"
import { TaskServiceLive } from "$services/TaskService"
import { ProjectServiceLive } from "$services/ProjectService"
import { WebSocketServiceLive } from "$services/WebSocketService"

// Infrastructure layer (no dependencies)
const InfrastructureLayer = Layer.merge(
  ApiClientLive,
  WebSocketServiceLive
)

// Service layer (depends on infrastructure)
const ServiceLayer = Layer.mergeAll(
  TaskServiceLive,
  ProjectServiceLive
).pipe(
  Layer.provide(InfrastructureLayer)
)

// Complete application layer
export const AppLayer = ServiceLayer
```

### What's Happening?

1. **`Layer.merge`**: Combines layers that can initialize in parallel
2. **`Layer.mergeAll`**: Same as merge, but for multiple layers
3. **`Layer.provide`**: Satisfies layer dependencies

---

## Runtime Configuration

### Create Runtime

Create `src/lib/effects/runtime.ts`:

```typescript
import { Effect, Layer, Runtime } from "effect"
import { AppLayer } from "./layers"

// Create runtime with all services
export const AppRuntime = Layer.toRuntime(AppLayer).pipe(
  Effect.runSync
)
```

### What is a Runtime?

A Runtime is a configured Effect execution environment:

- Contains all services
- Manages fiber execution
- Handles interruptions
- Provides configuration

---

## Using the Runtime

### In Svelte Components

```svelte
<script lang="ts">
  import { Effect } from "effect"
  import { AppRuntime } from "$effects/runtime"
  import { TaskService } from "$services/TaskService"

  let tasks = $state([])

  $effect(() => {
    Effect.runPromise(
      Effect.gen(function* () {
        const service = yield* TaskService
        return yield* service.listTasks()
      }),
      { runtime: AppRuntime }
    ).then(loadedTasks => {
      tasks = loadedTasks
    })
  })
</script>
```

### In SvelteKit Load Functions

```typescript
// +page.ts
import { Effect } from "effect"
import { AppRuntime } from "$effects/runtime"
import { TaskService } from "$services/TaskService"

export const load = async () => {
  const tasks = await Effect.runPromise(
    Effect.gen(function* () {
      const service = yield* TaskService
      return yield* service.listTasks()
    }),
    { runtime: AppRuntime }
  )

  return { tasks }
}
```

### In API Routes

```typescript
// +server.ts
import { Effect } from "effect"
import { json } from "@sveltejs/kit"
import { AppRuntime } from "$effects/runtime"
import { TaskService } from "$services/TaskService"

export const GET = async () => {
  const tasks = await Effect.runPromise(
    Effect.gen(function* () {
      const service = yield* TaskService
      return yield* service.listTasks()
    }),
    { runtime: AppRuntime }
  )

  return json(tasks)
}
```

---

## Environment-Specific Layers

### Development vs Production

```typescript
// layers.ts
const ApiClientDev = Layer.succeed(
  ApiClient,
  makeApiClient({
    baseUrl: "http://localhost:3000/api",
    timeout: 30000
  })
)

const ApiClientProd = Layer.succeed(
  ApiClient,
  makeApiClient({
    baseUrl: "https://api.taskmaster.com",
    timeout: 10000
  })
)

// Choose based on environment
const ApiClientLive =
  import.meta.env.MODE === "production"
    ? ApiClientProd
    : ApiClientDev

// Use in infrastructure layer
const InfrastructureLayer = Layer.merge(
  ApiClientLive,
  WebSocketServiceLive
)
```

---

## Testing with Layers

### Test Layer

```typescript
// Create test layer with mocks
import { TestApiClient } from "$services/ApiClient.test"

export const TestLayer = Layer.mergeAll(
  TaskServiceLive,
  ProjectServiceLive
).pipe(
  Layer.provide(TestApiClient)
)
```

### Use in Tests

```typescript
import { Effect } from "effect"
import { TestLayer } from "$effects/test-layers"
import { TaskService } from "$services/TaskService"

describe('Task workflow', () => {
  it('creates and lists tasks', async () => {
    const program = Effect.gen(function* () {
      const service = yield* TaskService

      yield* service.createTask({
        title: "Test Task",
        priority: "high",
        projectId: "test-project"
      })

      const tasks = yield* service.listTasks()
      return tasks
    })

    const tasks = await Effect.runPromise(
      program.pipe(Effect.provide(TestLayer))
    )

    expect(tasks).toHaveLength(1)
  })
})
```

---

## Advanced Patterns

### Layer Memoization

Layers are memoized by default:

```typescript
// This service is built only once
const ExpensiveServiceLive = Layer.effect(
  ExpensiveService,
  Effect.gen(function* () {
    console.log("Building service") // Runs once
    yield* Effect.sleep("5 seconds")
    return makeService()
  })
)
```

### Fresh Layers

Force a new instance:

```typescript
const AlwaysNewService = Layer.effect(
  Service,
  makeService
).pipe(Layer.fresh)
```

### Dynamic Layers

Create layers at runtime:

```typescript
const makeUserLayer = (userId: string) =>
  Layer.effect(
    UserContext,
    Effect.succeed({ userId, permissions: [] })
  )

// Use with specific user
const program = pipe(
  userWorkflow,
  Effect.provide(makeUserLayer("user-123"))
)
```

---

## Layer Checklist

Before moving to Chapter 5, verify:

- ✅ Infrastructure layer created
- ✅ Service layer composes services
- ✅ Application layer exports all services
- ✅ Runtime created and exported
- ✅ Runtime used in components
- ✅ Test layers configured

---

## Common Issues

### Issue: Circular Dependencies

```typescript
// ❌ Bad - circular dependency
const ServiceA = Layer.effect(A, Effect.gen(function* () {
  const b = yield* B  // Depends on B
  return makeA(b)
}))

const ServiceB = Layer.effect(B, Effect.gen(function* () {
  const a = yield* A  // Depends on A
  return makeB(a)
}))
```

**Solution**: Refactor to remove cycle or introduce intermediary.

### Issue: Missing Dependency

```typescript
// Error: TaskService requires ApiClient
const AppLayer = TaskServiceLive
```

**Solution**: Provide all dependencies:

```typescript
const AppLayer = TaskServiceLive.pipe(
  Layer.provide(ApiClientLive)
)
```

---

## What We Accomplished

In this chapter, we:

1. ✅ Understood Layer concepts
2. ✅ Created infrastructure layer
3. ✅ Composed service layer
4. ✅ Built application layer
5. ✅ Created runtime
6. ✅ Learned to use runtime in components

---

## Next Steps

With our runtime configured, we'll move to [Chapter 5: Components & UI](../05-components/README.md) where we'll:

- Create reusable Effect-Svelte utilities
- Build TaskList component
- Implement TaskForm with validation
- Create Dashboard with analytics
- Handle real-time updates

[Continue to Chapter 5 →](../05-components/README.md)

---

[← Chapter 3](../03-services/README.md) | [Main](../README.md) | [Chapter 5 →](../05-components/README.md)
