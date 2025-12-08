# EffectTS Quick Reference Guide

**Essential Patterns and Code Snippets**

---

## Table of Contents

1. [Creating Effects](#creating-effects)
2. [Transforming Effects](#transforming-effects)
3. [Error Handling](#error-handling)
4. [Combining Effects](#combining-effects)
5. [Context & Services](#context--services)
6. [React Integration](#react-integration)
7. [Svelte Integration](#svelte-integration)
8. [Common Patterns](#common-patterns)
9. [Testing](#testing)
10. [Performance](#performance)

---

## Creating Effects

### From Values

```typescript
import { Effect } from "effect"

// Pure success
const success = Effect.succeed(42)

// Pure failure
const failure = Effect.fail("error")

// Lazy computation
const lazy = Effect.sync(() => {
  console.log("Executing")
  return 42
})
```

### From Promises

```typescript
// Simple promise
const fromPromise = Effect.promise(() => 
  fetch("/api/data").then(r => r.json())
)

// With error handling
const withErrorHandling = Effect.tryPromise({
  try: () => fetch("/api/data").then(r => r.json()),
  catch: (error) => ({ _tag: "FetchError" as const, error })
})
```

### From Throwing Functions

```typescript
// Catch exceptions
const parseJSON = (text: string) =>
  Effect.try({
    try: () => JSON.parse(text),
    catch: (error) => ({ _tag: "ParseError" as const, message: String(error) })
  })
```

### Async Functions

```typescript
const asyncEffect = Effect.async<number, string>((resume) => {
  setTimeout(() => {
    resume(Effect.succeed(42))
  }, 1000)
})
```

---

[↑ Back to Top](#table-of-contents)

## Transforming Effects

### Map (Transform Success)

```typescript
import { pipe } from "effect"

// Using pipe
const doubled = pipe(
  Effect.succeed(21),
  Effect.map(x => x * 2)
)

// Using method
const doubled2 = Effect.succeed(21).pipe(
  Effect.map(x => x * 2)
)
```

### FlatMap (Chain Effects)

```typescript
const fetchAndParse = pipe(
  fetchData(),
  Effect.flatMap(data => parseData(data)),
  Effect.flatMap(parsed => saveData(parsed))
)
```

### Tap (Side Effects)

```typescript
import { Console } from "effect"

const withLogging = pipe(
  fetchUser("123"),
  Effect.tap(user => Console.log(`Fetched: ${user.name}`)),
  Effect.map(user => user.email)
)
```

### MapError (Transform Errors)

```typescript
const normalized = pipe(
  riskyOperation(),
  Effect.mapError(error => ({
    _tag: "OperationFailed" as const,
    original: error
  }))
)
```

---

[↑ Back to Top](#table-of-contents)

## Error Handling

### Catch All Errors

```typescript
const withFallback = pipe(
  riskyEffect,
  Effect.catchAll(error => Effect.succeed(defaultValue))
)
```

### Catch Specific Errors

```typescript
type AppError = 
  | { _tag: "NotFound" }
  | { _tag: "NetworkError" }

const handled = pipe(
  fetchData(),
  Effect.catchTag("NotFound", () => Effect.succeed(defaultData)),
  Effect.catchTag("NetworkError", error => retryFetch())
)
```

### Retry with Schedule

```typescript
import { Schedule } from "effect"

const resilient = pipe(
  fetchData(),
  Effect.retry({
    times: 3,
    schedule: Schedule.exponential("100 millis")
  })
)
```

### Convert to Option

```typescript
const optional = pipe(
  riskyEffect,
  Effect.option // Effect<Option<A>, never>
)
```

### Provide Fallback

```typescript
const withDefault = pipe(
  fetchConfig(),
  Effect.orElse(() => Effect.succeed(defaultConfig))
)
```

---

[↑ Back to Top](#table-of-contents)

## Combining Effects

### Sequential (All)

```typescript
// Object
const result = Effect.all({
  user: fetchUser(id),
  posts: fetchPosts(id),
  stats: fetchStats(id)
})

// Array
const results = Effect.all([
  effect1,
  effect2,
  effect3
])
```

### Parallel

```typescript
const concurrent = Effect.all(
  [fetchA(), fetchB(), fetchC()],
  { concurrency: "unbounded" }
)

// Controlled concurrency
const limited = Effect.all(
  items.map(processItem),
  { concurrency: 5 }
)
```

### Race

```typescript
// First to complete
const fastest = Effect.race(
  fetchFromServerA(),
  fetchFromServerB()
)

// With timeout
const withTimeout = Effect.race(
  fetchData(),
  Effect.sleep("5 seconds").pipe(
    Effect.flatMap(() => Effect.fail({ _tag: "Timeout" }))
  )
)
```

### Zip

```typescript
// Combine two effects
const zipped = Effect.zip(effect1, effect2)

// With custom combiner
const combined = pipe(
  Effect.zip(effect1, effect2),
  Effect.map(([a, b]) => combine(a, b))
)
```

---

[↑ Back to Top](#table-of-contents)

## Context & Services

### Define Service

```typescript
import { Context } from "effect"

interface Logger {
  readonly log: (message: string) => Effect.Effect<void>
}

const Logger = Context.GenericTag<Logger>("Logger")
```

### Use Service

```typescript
// Pipe style
const program = pipe(
  Logger,
  Effect.flatMap(logger => logger.log("Hello"))
)
```

### Provide Service

```typescript
const implementation: Logger = {
  log: (message) => Console.log(message)
}

const program = pipe(
  myEffect,
  Effect.provideService(Logger, implementation)
)
```

### Create Layer

```typescript
import { Layer } from "effect"

const LoggerLive = Layer.succeed(Logger, {
  log: (message) => Console.log(message)
})

// Layer with dependencies
const ApiClientLive = Layer.effect(
  ApiClient,
  pipe(
    Config,
    Effect.map(config => makeApiClient(config))
  )
)
```

### Compose Layers

```typescript
const AppLayer = Layer.mergeAll(
  LoggerLive,
  ApiClientLive,
  DatabaseLive
)

const program = pipe(
  myEffect,
  Effect.provide(AppLayer)
)
```

---

[↑ Back to Top](#table-of-contents)

## React Integration

### Setup Runtime

```typescript
import { Runtime } from "effect"
import { createContext, useContext } from "react"

const RuntimeContext = createContext<Runtime.Runtime<never> | null>(null)

export const RuntimeProvider = ({ children }) => (
  <RuntimeContext.Provider value={Runtime.defaultRuntime}>
    {children}
  </RuntimeContext.Provider>
)

export const useRuntime = () => {
  const runtime = useContext(RuntimeContext)
  if (!runtime) throw new Error("Runtime not provided")
  return runtime
}
```

### Effect-RX Basics

```typescript
import { Rx, useRx } from "@effect-rx/rx-react"

// Create reactive value
const counter = Rx.make(0)

// Use in component
function Counter() {
  const [count, setCount] = useRx(counter)
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  )
}
```

### Fetch Data with Rx

```typescript
const userRx = (userId: string) =>
  Rx.make(() => fetchUser(userId))

function UserProfile({ userId }: { userId: string }) {
  const user = useRx(userRx(userId))
  
  if (user._tag === "loading") return <Spinner />
  if (user._tag === "error") return <Error error={user.error} />
  
  return <div>{user.value.name}</div>
}
```

### Custom Hook

```typescript
import { useState, useEffect } from "react"
import { Effect, Exit } from "effect"

function useEffect<A, E>(effect: Effect.Effect<A, E>) {
  const [state, setState] = useState<
    | { _tag: "loading" }
    | { _tag: "success"; data: A }
    | { _tag: "error"; error: E }
  >({ _tag: "loading" })
  
  useEffect(() => {
    Effect.runPromiseExit(effect).then(exit => {
      if (Exit.isSuccess(exit)) {
        setState({ _tag: "success", data: exit.value })
      } else {
        setState({ _tag: "error", error: exit.cause })
      }
    })
  }, [])
  
  return state
}
```

---

[↑ Back to Top](#table-of-contents)

## Svelte Integration

### Effect Store

```typescript
import { writable } from "svelte/store"
import { Effect, Exit } from "effect"

export function effectStore<A, E>(effect: Effect.Effect<A, E>) {
  const store = writable<
    | { _tag: "loading" }
    | { _tag: "success"; data: A }
    | { _tag: "error"; error: E }
  >({ _tag: "loading" })
  
  Effect.runPromiseExit(effect).then(exit => {
    if (Exit.isSuccess(exit)) {
      store.set({ _tag: "success", data: exit.value })
    } else {
      store.set({ _tag: "error", error: exit.cause })
    }
  })
  
  return store
}
```

### Use in Component

```svelte
<script lang="ts">
  import { effectStore } from "$lib/effect-store"
  import { fetchUser } from "$lib/api"
  
  export let userId: string
  
  $: user = effectStore(fetchUser(userId))
</script>

{#if $user._tag === "loading"}
  <p>Loading...</p>
{:else if $user._tag === "error"}
  <p>Error: {$user.error}</p>
{:else}
  <p>{$user.data.name}</p>
{/if}
```

### Derived Store

```typescript
import { derived } from "svelte/store"

const derived Store = derived(
  baseStore,
  ($base, set) => {
    Effect.runPromise(transform($base)).then(set)
  }
)
```

---

[↑ Back to Top](#table-of-contents)

## Common Patterns

### Pipeline Pattern

```typescript
const result = pipe(
  input,
  step1,
  step2,
  step3
)
```

### Pipe Pattern

```typescript
const program = pipe(
  effect1,
  Effect.flatMap(a =>
    pipe(
      effect2,
      Effect.map(b => combine(a, b))
    )
  )
)
```

### Conditional Logic

```typescript
const result = Effect.if(condition, {
  onTrue: () => effect1,
  onFalse: () => effect2
})
```

### Resource Management

```typescript
const withResource = Effect.acquireUseRelease(
  acquire, // Acquire resource
  use,     // Use resource
  release  // Release resource (always runs)
)
```

### Retry Pattern

```typescript
import { Schedule } from "effect"

const retry = pipe(
  riskyEffect,
  Effect.retry({
    while: (error) => error._tag === "Temporary",
    times: 3,
    schedule: Schedule.exponential("100 millis")
  })
)
```

### Timeout Pattern

```typescript
const withTimeout = pipe(
  longRunningEffect,
  Effect.timeout("30 seconds"),
  Effect.catchTag("TimeoutException", () => 
    Effect.succeed(defaultValue)
  )
)
```

### Debounce Pattern

```typescript
const debounced = pipe(
  Effect.sleep("500 millis"),
  Effect.flatMap(() => actualEffect)
)
```

---

[↑ Back to Top](#table-of-contents)

## Testing

### Test Pure Effects

```typescript
import { Effect, Exit } from "effect"
import { describe, it, expect } from "vitest"

describe("validation", () => {
  it("should validate correct data", async () => {
    const result = await Effect.runPromiseExit(
      validateData(validInput)
    )
    
    expect(Exit.isSuccess(result)).toBe(true)
  })
})
```

### Mock Services

```typescript
const MockLogger: Logger = {
  log: (message) => Console.log(`[MOCK] ${message}`)
}

const MockLayer = Layer.succeed(Logger, MockLogger)

const test = pipe(
  programUnderTest,
  Effect.provide(MockLayer)
)
```

### Test with Pipe

```typescript
const test = pipe(
  effectUnderTest,
  Effect.map(result => {
    expect(result).toEqual(expected)
    return result
  })
)

Effect.runPromise(test)
```

---

[↑ Back to Top](#table-of-contents)

## Performance

### Batching Requests

```typescript
import { Request, RequestResolver } from "effect"

interface GetUser extends Request.Request<User, UserNotFound> {
  readonly _tag: "GetUser"
  readonly id: string
}

const UserResolver = RequestResolver.makeBatched(
  (requests: GetUser[]) => batchFetch(requests.map(r => r.id))
)

const getUser = (id: string) =>
  Effect.request(GetUser({ id }), UserResolver)
```

### Caching

```typescript
import { Cache, Duration } from "effect"

const cache = Cache.make({
  capacity: 100,
  timeToLive: Duration.minutes(5),
  lookup: (key: string) => expensiveOperation(key)
})
```

### Memoization

```typescript
const memoized = Effect.cached(expensiveEffect)

// Multiple calls will return cached result
const result1 = yield* memoized
const result2 = yield* memoized // Uses cache
```

### Lazy Loading

```typescript
const lazyData = Effect.suspend(() => 
  shouldLoad ? fetchData() : Effect.succeed(null)
)
```

---

[↑ Back to Top](#table-of-contents)

## Type Patterns

### Discriminated Unions

```typescript
type Result<T, E> =
  | { _tag: "success"; value: T }
  | { _tag: "error"; error: E }

const handleResult = <T, E>(result: Result<T, E>) => {
  switch (result._tag) {
    case "success":
      return result.value
    case "error":
      return handleError(result.error)
  }
}
```

### Branded Types

```typescript
type UserId = string & { readonly _brand: "UserId" }
type PostId = string & { readonly _brand: "PostId" }

const UserId = (id: string): UserId => id as UserId

// Type-safe IDs
const getUser = (id: UserId) => fetchUser(id)
```

### Effect Type Aliases

```typescript
// Common effect signatures
type AsyncEffect<A> = Effect.Effect<A, never, never>
type FailableEffect<A, E> = Effect.Effect<A, E, never>
type ServiceEffect<A, R> = Effect.Effect<A, never, R>
```

---

[↑ Back to Top](#table-of-contents)

## Debugging

### Add Logging

```typescript
import { Console } from "effect"

const withLogging = pipe(
  effect,
  Effect.tap(() => Console.log("Step completed")),
  Effect.tapError(error => Console.error("Failed:", error))
)
```

### Annotate Errors

```typescript
const annotated = pipe(
  effect,
  Effect.annotateCurrentSpan("userId", userId),
  Effect.annotateCurrentSpan("operation", "fetchUser")
)
```

### Inspect Values

```typescript
const inspected = pipe(
  effect,
  Effect.tap(value => Effect.sync(() => console.log("Value:", value)))
)
```

---

[↑ Back to Top](#table-of-contents)

## Common Imports

```typescript
// Core
import { Effect, pipe } from "effect"

// Error handling
import { Exit, Cause } from "effect"

// Context
import { Context, Layer } from "effect"

// State
import { Ref } from "effect"

// Async
import { Deferred, Fiber } from "effect"

// Collections
import { Array, Option, Either } from "effect"

// Scheduling
import { Schedule, Duration } from "effect"

// Caching
import { Cache } from "effect"

// Streams
import { Stream, Sink } from "effect"

// React
import { Rx, useRx } from "@effect-rx/rx-react"

// Console
import { Console } from "effect"

// Schema
import { Schema } from "@effect/schema"
```

---

[↑ Back to Top](#table-of-contents)

## Quick Tips

1. **Use `pipe` for readability** - It makes data flow clear
2. **Type your errors** - Use discriminated unions with `_tag`
3. **Leverage Context** - Avoid passing dependencies manually
4. **Test with mocks** - Create mock layers for testing
5. **Handle errors early** - Use `catchTag` for specific error handling
6. **Compose small effects** - Build complex logic from simple pieces
7. **Use generators for clarity** - When you have many sequential steps
8. **Cache expensive operations** - Use `Cache` for repeated computations
9. **Batch requests** - Use `RequestResolver` for N+1 problems
10. **Keep effects pure** - Describe, don't execute

---

[↑ Back to Top](#table-of-contents)

## Resources

- **Documentation**: https://effect.website
- **API Docs**: https://effect-ts.github.io/effect
- **Discord**: https://discord.gg/effect-ts
- **Examples**: https://github.com/Effect-TS/effect/tree/main/packages/effect/examples

---

This quick reference covers the most common patterns you'll use daily with Effect-TS. Keep it handy for quick lookups!
