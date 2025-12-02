# EffectTS Frontend Mastery Course

**A Comprehensive Guide for TypeScript Developers**  
*Leveraging Effect-TS in React and Svelte Applications*

---

## Course Overview

This course teaches you to master Effect-TS for building robust, type-safe frontend applications. Effect-TS brings functional effect systems to TypeScript, enabling you to handle async operations, errors, dependencies, and side effects with compositional elegance.

**Prerequisites:**
- Strong TypeScript knowledge (generics, advanced types, conditional types)
- Experience with React or Svelte
- Familiarity with functional programming concepts (composition, HOFs, pipelines)

**What You'll Build:**
- Form validation system with composable validators
- Data fetching layer with retries, caching, and error handling
- Real-time notification system
- Feature flag service with A/B testing
- Production-ready admin dashboard

---

## Module 1: Effect Fundamentals

### 1.1 The Effect Type System

Effect-TS centers on the `Effect<Success, Error, Requirements>` type, representing computations that:
- May succeed with type `Success`
- May fail with type `Error`
- Require dependencies of type `Requirements`

```typescript
import { Effect, Console } from "effect"

// Effect<void, never, never>
// Success: void, Error: never (can't fail), Requirements: never (no deps)
const greet = Console.log("Hello, Effect!")

// Effect<number, never, never>
const computation = Effect.succeed(42)

// Effect<never, string, never>
const failure = Effect.fail("Something went wrong")

// Sync computation with error handling
const divide = (a: number, b: number): Effect.Effect<number, string, never> =>
  b === 0 
    ? Effect.fail("Division by zero")
    : Effect.succeed(a / b)
```

**Key Insight:** Effects are descriptions of computations, not the computations themselves. They're lazy and only execute when you run them.

### 1.2 Creating Effects

```typescript
import { Effect } from "effect"

// From pure values
const pure = Effect.succeed(100)

// From sync functions
const syncEffect = Effect.sync(() => {
  console.log("Computing...")
  return Math.random()
})

// From promises
const fetchUser = (id: string) =>
  Effect.promise(() => 
    fetch(`/api/users/${id}`).then(res => res.json())
  )

// From async functions
const asyncEffect = Effect.tryPromise({
  try: async () => {
    const response = await fetch("/api/data")
    return response.json()
  },
  catch: (error) => ({
    _tag: "NetworkError" as const,
    error
  })
})

// From potentially throwing functions
const parseJSON = (text: string) =>
  Effect.try({
    try: () => JSON.parse(text),
    catch: (error) => ({
      _tag: "ParseError" as const,
      message: String(error)
    })
  })
```

### 1.3 Running Effects

```typescript
import { Effect, Runtime } from "effect"

// Basic execution (for never-failing effects)
const program = Effect.sync(() => console.log("Running!"))
Effect.runSync(program)

// With promise for async effects
const asyncProgram = Effect.promise(() => fetch("/api"))
await Effect.runPromise(asyncProgram)

// With error handling
const riskyProgram = divide(10, 0)

await Effect.runPromise(
  riskyProgram.pipe(
    Effect.catchAll(error => 
      Effect.sync(() => console.error("Caught:", error))
    )
  )
)

// Exit handling for complete control
const result = await Effect.runPromiseExit(riskyProgram)

if (result._tag === "Success") {
  console.log("Value:", result.value)
} else {
  console.error("Failed:", result.cause)
}
```

### 1.4 Transforming Effects with Pipe

The `pipe` function is your primary composition tool, enabling clean data transformation pipelines:

```typescript
import { Effect, pipe } from "effect"

const result = pipe(
  Effect.succeed(5),
  Effect.map(x => x * 2),        // 10
  Effect.flatMap(x => 
    x > 8 
      ? Effect.succeed(x) 
      : Effect.fail("Too small")
  ),
  Effect.map(x => `Result: ${x}`)
)

// Alternative: method chaining (use whichever you prefer)
const result2 = Effect.succeed(5)
  .pipe(Effect.map(x => x * 2))
  .pipe(Effect.flatMap(x => 
    x > 8 
      ? Effect.succeed(x) 
      : Effect.fail("Too small")
  ))
  .pipe(Effect.map(x => `Result: ${x}`))
```

### 1.5 Error Handling Patterns

```typescript
import { Effect, pipe, Exit } from "effect"

// Typed errors with discriminated unions
type AppError = 
  | { _tag: "NetworkError"; status: number }
  | { _tag: "ParseError"; message: string }
  | { _tag: "NotFound" }

const fetchAndParse = (url: string): Effect.Effect<unknown, AppError, never> =>
  pipe(
    Effect.tryPromise({
      try: () => fetch(url),
      catch: () => ({ _tag: "NetworkError" as const, status: 500 })
    }),
    Effect.flatMap(response => 
      response.ok
        ? Effect.promise(() => response.json()).pipe(
            Effect.catchAll(() => 
              Effect.fail({ _tag: "ParseError" as const, message: "Invalid JSON" })
            )
          )
        : Effect.fail({ _tag: "NotFound" as const })
    )
  )

// Catching specific errors
const withFallback = pipe(
  fetchAndParse("/api/data"),
  Effect.catchTag("NetworkError", error => 
    Effect.succeed({ fallback: true, error })
  ),
  Effect.catchTag("ParseError", () => 
    Effect.succeed({ data: [] })
  )
)

// Retrying with exponential backoff
const resilient = pipe(
  fetchAndParse("/api/data"),
  Effect.retry({
    times: 3,
    schedule: Schedule.exponential("100 millis")
  })
)

// Converting errors to optional values
const optional = pipe(
  fetchAndParse("/api/data"),
  Effect.option // Effect<Option<Data>, never, never>
)
```

---

## Module 2: Composition and Combinators

### 2.1 Sequential Composition

```typescript
import { Effect, pipe } from "effect"

// map: transform success values (functor)
const doubled = pipe(
  Effect.succeed(21),
  Effect.map(x => x * 2)
)

// flatMap: chain dependent computations (monad)
const validateAndSave = (input: string) =>
  pipe(
    parseInput(input),
    Effect.flatMap(validated => 
      saveToDatabase(validated)
    ),
    Effect.flatMap(saved => 
      sendNotification(saved.id)
    )
  )

// tap: perform side effects without changing the value
const withLogging = pipe(
  fetchUser("123"),
  Effect.tap(user => Console.log(`Fetched: ${user.name}`)),
  Effect.map(user => user.email)
)

// all: run effects in sequence, collect results
const userProfile = Effect.all([
  fetchUser(userId),
  fetchPosts(userId),
  fetchFollowers(userId)
]).pipe(
  Effect.map(([user, posts, followers]) => ({
    user,
    posts,
    followers
  }))
)
```

### 2.2 Parallel Composition

```typescript
import { Effect } from "effect"

// Concurrent execution
const dashboard = Effect.all(
  [
    fetchUser(userId),
    fetchStats(),
    fetchNotifications()
  ],
  { concurrency: "unbounded" } // Run all concurrently
).pipe(
  Effect.map(([user, stats, notifications]) => ({
    user,
    stats,
    notifications
  }))
)

// Controlled concurrency
const processItems = (items: string[]) =>
  Effect.all(
    items.map(item => processItem(item)),
    { concurrency: 5 } // Max 5 concurrent operations
  )

// Racing effects
const fetchWithTimeout = Effect.race(
  fetchData(),
  Effect.sleep("5 seconds").pipe(
    Effect.flatMap(() => Effect.fail({ _tag: "Timeout" as const }))
  )
)

// First successful result
const fetchFromMirrors = Effect.raceAll([
  fetch("https://api1.example.com"),
  fetch("https://api2.example.com"),
  fetch("https://api3.example.com")
].map(Effect.promise))
```

### 2.3 Conditional Logic

```typescript
import { Effect, pipe } from "effect"

// if/else with Effect.if
const processUser = (user: User) =>
  Effect.if(user.isActive, {
    onTrue: () => activateAccount(user),
    onFalse: () => sendReactivationEmail(user)
  })

// Pattern matching with matchEffect
const handleResponse = (status: number) =>
  pipe(
    status,
    Effect.matchEffect({
      onFailure: (code) => 
        code === 404 
          ? Effect.succeed({ notFound: true })
          : Effect.fail({ code, message: "Request failed" }),
      onSuccess: (code) => 
        fetchResponseData()
    })
  )

// when/unless for conditional execution
const audit = (action: string) =>
  pipe(
    performAction(action),
    Effect.tap(result =>
      Effect.when(
        isProduction(),
        () => logToAuditTrail(action, result)
      )
    )
  )
```

### 2.4 Building Composable Pipelines

```typescript
import { Effect, pipe, flow } from "effect"

// Domain types
type RawInput = { text: string; userId: string }
type ValidatedInput = { content: string; userId: string; wordCount: number }
type SavedPost = ValidatedInput & { id: string; timestamp: number }

// Composable operations
const validate = (input: RawInput): Effect.Effect<ValidatedInput, ValidationError> =>
  input.text.trim().length === 0
    ? Effect.fail({ _tag: "EmptyContent" as const })
    : input.text.length > 1000
    ? Effect.fail({ _tag: "TooLong" as const, max: 1000 })
    : Effect.succeed({
        content: input.text.trim(),
        userId: input.userId,
        wordCount: input.text.split(/\s+/).length
      })

const save = (data: ValidatedInput): Effect.Effect<SavedPost, DatabaseError> =>
  Effect.tryPromise({
    try: async () => {
      const response = await fetch("/api/posts", {
        method: "POST",
        body: JSON.stringify(data)
      })
      return response.json()
    },
    catch: () => ({ _tag: "SaveFailed" as const })
  })

const notify = (post: SavedPost): Effect.Effect<void, NotificationError> =>
  Effect.tryPromise({
    try: () => 
      fetch("/api/notify", {
        method: "POST",
        body: JSON.stringify({ postId: post.id })
      }),
    catch: () => ({ _tag: "NotificationFailed" as const })
  }).pipe(Effect.asVoid)

// Compose into pipeline
const createPost = (input: RawInput) =>
  pipe(
    validate(input),
    Effect.flatMap(save),
    Effect.tap(notify),
    Effect.tapError(error => 
      Console.error("Post creation failed:", error)
    )
  )
```

---

## Module 3: Dependency Injection with Context

### 3.1 Understanding Context and Services

Effect's Context system provides type-safe dependency injection without globals or props drilling:

```typescript
import { Effect, Context } from "effect"

// Define a service interface
interface Logger {
  readonly log: (message: string) => Effect.Effect<void>
  readonly error: (message: string) => Effect.Effect<void>
}

// Create a Tag (service identifier)
const Logger = Context.GenericTag<Logger>("Logger")

// Use the service
const greetUser = (name: string) =>
  Effect.gen(function* () {
    const logger = yield* Logger
    yield* logger.log(`Hello, ${name}!`)
    return `Greeted ${name}`
  })

// Alternative with pipe
const greetUserPipe = (name: string) =>
  pipe(
    Logger,
    Effect.flatMap(logger => logger.log(`Hello, ${name}!`)),
    Effect.map(() => `Greeted ${name}`)
  )

// Provide implementation
const ConsoleLogger: Logger = {
  log: (message) => Console.log(message),
  error: (message) => Console.error(message)
}

const program = pipe(
  greetUser("Alice"),
  Effect.provideService(Logger, ConsoleLogger)
)

await Effect.runPromise(program)
```

### 3.2 Building a Service Layer

```typescript
import { Effect, Context, Layer } from "effect"

// API Service
interface ApiClient {
  readonly get: <T>(url: string) => Effect.Effect<T, ApiError>
  readonly post: <T, B>(url: string, body: B) => Effect.Effect<T, ApiError>
}

const ApiClient = Context.GenericTag<ApiClient>("ApiClient")

type ApiError = 
  | { _tag: "NetworkError"; cause: unknown }
  | { _tag: "InvalidResponse"; status: number }

// Config Service
interface AppConfig {
  readonly apiBaseUrl: string
  readonly timeout: number
}

const AppConfig = Context.GenericTag<AppConfig>("AppConfig")

// Implementation that depends on config
const makeApiClient = (config: AppConfig): ApiClient => ({
  get: <T>(url: string) =>
    Effect.tryPromise({
      try: async () => {
        const controller = new AbortController()
        const timeout = setTimeout(() => controller.abort(), config.timeout)
        
        const response = await fetch(`${config.apiBaseUrl}${url}`, {
          signal: controller.signal
        })
        
        clearTimeout(timeout)
        
        if (!response.ok) {
          throw { status: response.status }
        }
        
        return response.json() as Promise<T>
      },
      catch: (error) => 
        typeof error === "object" && error !== null && "status" in error
          ? { _tag: "InvalidResponse" as const, status: error.status as number }
          : { _tag: "NetworkError" as const, cause: error }
    }),
  
  post: <T, B>(url: string, body: B) =>
    Effect.tryPromise({
      try: async () => {
        const response = await fetch(`${config.apiBaseUrl}${url}`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify(body)
        })
        
        if (!response.ok) {
          throw { status: response.status }
        }
        
        return response.json() as Promise<T>
      },
      catch: (error) =>
        typeof error === "object" && error !== null && "status" in error
          ? { _tag: "InvalidResponse" as const, status: error.status as number }
          : { _tag: "NetworkError" as const, cause: error }
    })
})

// Create a Layer (wires up dependencies)
const ApiClientLive = Layer.effect(
  ApiClient,
  Effect.gen(function* () {
    const config = yield* AppConfig
    return makeApiClient(config)
  })
)

// Usage in application code
const fetchUser = (id: string) =>
  Effect.gen(function* () {
    const api = yield* ApiClient
    return yield* api.get<User>(`/users/${id}`)
  })

// Provide all dependencies
const ConfigLive = Layer.succeed(AppConfig, {
  apiBaseUrl: "https://api.example.com",
  timeout: 5000
})

const program = pipe(
  fetchUser("123"),
  Effect.provide(ApiClientLive),
  Effect.provide(ConfigLive)
)

// Or compose layers
const AppLayer = ApiClientLive.pipe(
  Layer.provide(ConfigLive)
)

const programWithLayer = pipe(
  fetchUser("123"),
  Effect.provide(AppLayer)
)
```

### 3.3 Testing with Mock Services

```typescript
import { Effect, Context, Layer } from "effect"

// Mock API for testing
const MockApiClient: ApiClient = {
  get: <T>(url: string) =>
    Effect.succeed({
      id: "123",
      name: "Test User",
      email: "test@example.com"
    } as T),
  
  post: <T, B>(url: string, body: B) =>
    Effect.succeed({ success: true } as T)
}

const MockApiClientLayer = Layer.succeed(ApiClient, MockApiClient)

// Test your code with mocks
const testProgram = pipe(
  fetchUser("123"),
  Effect.provide(MockApiClientLayer)
)

await Effect.runPromise(testProgram) // Uses mock implementation
```

### 3.4 Advanced Service Patterns

```typescript
import { Effect, Context, Layer, Ref } from "effect"

// Service with state
interface UserCache {
  readonly get: (id: string) => Effect.Effect<Option<User>, never>
  readonly set: (id: string, user: User) => Effect.Effect<void>
}

const UserCache = Context.GenericTag<UserCache>("UserCache")

const makeUserCache = Effect.gen(function* () {
  const cache = yield* Ref.make<Map<string, User>>(new Map())
  
  return UserCache.of({
    get: (id) =>
      pipe(
        Ref.get(cache),
        Effect.map(map => Option.fromNullable(map.get(id)))
      ),
    
    set: (id, user) =>
      Ref.update(cache, map => new Map(map).set(id, user))
  })
})

const UserCacheLive = Layer.effect(UserCache, makeUserCache)

// Service that depends on other services
interface UserRepository {
  readonly fetchUser: (id: string) => Effect.Effect<User, RepositoryError>
}

const UserRepository = Context.GenericTag<UserRepository>("UserRepository")

const makeUserRepository = Effect.gen(function* () {
  const api = yield* ApiClient
  const cache = yield* UserCache
  
  return UserRepository.of({
    fetchUser: (id) =>
      pipe(
        cache.get(id),
        Effect.flatMap(Option.match({
          onNone: () =>
            pipe(
              api.get<User>(`/users/${id}`),
              Effect.tap(user => cache.set(id, user)),
              Effect.mapError(() => ({ 
                _tag: "FetchFailed" as const,
                userId: id 
              }))
            ),
          onSome: (user) => Effect.succeed(user)
        }))
      )
  })
})

const UserRepositoryLive = Layer.effect(
  UserRepository,
  makeUserRepository
).pipe(
  Layer.provide(ApiClientLive),
  Layer.provide(UserCacheLive)
)
```

---

## Module 4: React Integration

### 4.1 Setting Up Effect with React

```typescript
// Install dependencies
// npm install effect @effect/platform @effect-rx/rx-react

import { Effect, Runtime } from "effect"
import { createContext, useContext, ReactNode } from "react"

// Create a runtime context
const RuntimeContext = createContext<Runtime.Runtime<never> | null>(null)

export const RuntimeProvider = ({ children }: { children: ReactNode }) => {
  const runtime = Runtime.defaultRuntime
  
  return (
    <RuntimeContext.Provider value={runtime}>
      {children}
    </RuntimeContext.Provider>
  )
}

export const useRuntime = () => {
  const runtime = useContext(RuntimeContext)
  if (!runtime) throw new Error("Runtime not provided")
  return runtime
}
```

### 4.2 Effect-RX for Reactive State

Effect-RX provides reactive primitives that integrate beautifully with React:

```typescript
import { Rx } from "@effect-rx/rx-react"
import { Effect } from "effect"

// Create a reactive registry
const registry = Rx.make()

// Define reactive values (Rx)
const counter = Rx.make(0) // Rx<number>

const doubled = Rx.map(counter, n => n * 2)

const counterWithEffect = Rx.make((get) => {
  const count = get(counter)
  return Effect.succeed(count * 2)
})

// Use in React components
import { useRx } from "@effect-rx/rx-react"

function Counter() {
  const [count, setCount] = useRx(counter)
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  )
}

function DoubledDisplay() {
  const value = useRx(doubled) // Automatically updates
  return <p>Doubled: {value}</p>
}
```

### 4.3 Custom Effect Hooks

```typescript
import { useState, useEffect } from "react"
import { Effect, Exit } from "effect"
import { useRuntime } from "./RuntimeProvider"

// Basic effect runner hook
export function useEffectOnce<A, E>(
  effect: Effect.Effect<A, E, never>
) {
  const [state, setState] = useState<
    | { _tag: "loading" }
    | { _tag: "success"; data: A }
    | { _tag: "error"; error: E }
  >({ _tag: "loading" })
  
  const runtime = useRuntime()
  
  useEffect(() => {
    const cancel = Effect.runPromiseExit(effect, { runtime }).then(exit => {
      if (Exit.isSuccess(exit)) {
        setState({ _tag: "success", data: exit.value })
      } else {
        setState({ _tag: "error", error: Exit.unannotate(exit.cause) })
      }
    })
    
    return () => {
      // Effect provides cancellation
      cancel.then(fiber => fiber.interrupt())
    }
  }, [])
  
  return state
}

// Usage
function UserProfile({ userId }: { userId: string }) {
  const state = useEffectOnce(fetchUser(userId))
  
  switch (state._tag) {
    case "loading":
      return <div>Loading...</div>
    case "error":
      return <div>Error: {JSON.stringify(state.error)}</div>
    case "success":
      return <div>User: {state.data.name}</div>
  }
}
```

### 4.4 Effect-RX Patterns

```typescript
import { Rx } from "@effect-rx/rx-react"
import { Effect, Schedule } from "effect"

// Async data fetching with Rx
const userRx = (userId: string) =>
  Rx.make((get) =>
    pipe(
      Effect.sleep("100 millis"), // Debounce
      Effect.flatMap(() => fetchUser(userId))
    )
  )

// Derived reactive values
const userEmailRx = (userId: string) =>
  Rx.map(userRx(userId), user => user.email)

// Refreshable data
const postsRx = Rx.family((userId: string) =>
  Rx.make((get) =>
    pipe(
      fetchPosts(userId),
      Effect.tap(posts => Console.log(`Fetched ${posts.length} posts`))
    )
  )
)

// Polling data
const liveStatsRx = Rx.make((get) =>
  pipe(
    fetchStats(),
    Effect.repeat(Schedule.fixed("5 seconds"))
  )
)

// Component with reactive data
function UserDashboard({ userId }: { userId: string }) {
  const user = useRx(userRx(userId))
  const posts = useRx(postsRx(userId))
  const stats = useRx(liveStatsRx)
  
  // Handle loading states from Rx
  if (user._tag === "loading") return <div>Loading user...</div>
  if (user._tag === "error") return <div>Error loading user</div>
  
  return (
    <div>
      <h1>{user.value.name}</h1>
      <p>{user.value.email}</p>
      {/* posts and stats also have loading/error states */}
    </div>
  )
}
```

### 4.5 Form Validation with Effect

```typescript
import { Effect, pipe } from "effect"
import { useState } from "react"

// Validation errors
type ValidationError = 
  | { _tag: "Required"; field: string }
  | { _tag: "TooShort"; field: string; min: number }
  | { _tag: "InvalidEmail"; field: string }

// Validators
const required = (value: string, field: string): Effect.Effect<string, ValidationError> =>
  value.trim().length === 0
    ? Effect.fail({ _tag: "Required", field })
    : Effect.succeed(value)

const minLength = (min: number) => (value: string, field: string): Effect.Effect<string, ValidationError> =>
  value.length < min
    ? Effect.fail({ _tag: "TooShort", field, min })
    : Effect.succeed(value)

const email = (value: string, field: string): Effect.Effect<string, ValidationError> =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)
    ? Effect.succeed(value)
    : Effect.fail({ _tag: "InvalidEmail", field })

// Compose validators
const validateField = (
  value: string,
  field: string,
  ...validators: Array<(v: string, f: string) => Effect.Effect<string, ValidationError>>
): Effect.Effect<string, ValidationError[]> =>
  pipe(
    validators.map(validator => validator(value, field)),
    Effect.all({ concurrency: "unbounded" }),
    Effect.map(() => value),
    Effect.mapError(errors => Array.isArray(errors) ? errors : [errors])
  )

// Form validation
interface SignupForm {
  email: string
  password: string
  username: string
}

const validateSignupForm = (form: SignupForm): Effect.Effect<SignupForm, ValidationError[]> =>
  Effect.all({
    email: validateField(form.email, "email", required, email),
    password: validateField(form.password, "password", required, minLength(8)),
    username: validateField(form.username, "username", required, minLength(3))
  }).pipe(
    Effect.map(() => form)
  )

// React component
function SignupForm() {
  const [form, setForm] = useState<SignupForm>({
    email: "",
    password: "",
    username: ""
  })
  const [errors, setErrors] = useState<ValidationError[]>([])
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    
    const result = await Effect.runPromiseExit(validateSignupForm(form))
    
    if (Exit.isSuccess(result)) {
      // Submit form
      console.log("Valid form:", result.value)
    } else {
      // Show errors
      const validationErrors = Exit.unannotate(result.cause)
      setErrors(Array.isArray(validationErrors) ? validationErrors : [validationErrors])
    }
  }
  
  const getFieldError = (field: string) =>
    errors.find(e => e.field === field)
  
  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          type="email"
          value={form.email}
          onChange={e => setForm({ ...form, email: e.target.value })}
          placeholder="Email"
        />
        {getFieldError("email") && (
          <span className="error">
            {getFieldError("email")?._tag === "Required" && "Email is required"}
            {getFieldError("email")?._tag === "InvalidEmail" && "Invalid email"}
          </span>
        )}
      </div>
      {/* Similar for password and username */}
      <button type="submit">Sign Up</button>
    </form>
  )
}
```

---

## Module 5: Svelte Integration

### 5.1 Effect Stores in Svelte

Svelte's reactivity pairs naturally with Effect's composability:

```typescript
// lib/effect-store.ts
import { writable, derived, type Readable } from "svelte/store"
import { Effect, Runtime, Exit } from "effect"

export function effectStore<A, E>(
  effect: Effect.Effect<A, E, never>,
  runtime: Runtime.Runtime<never> = Runtime.defaultRuntime
) {
  type State = 
    | { _tag: "loading" }
    | { _tag: "success"; data: A }
    | { _tag: "error"; error: E }
  
  const store = writable<State>({ _tag: "loading" })
  
  Effect.runPromiseExit(effect, { runtime }).then(exit => {
    if (Exit.isSuccess(exit)) {
      store.set({ _tag: "success", data: exit.value })
    } else {
      store.set({ _tag: "error", error: Exit.unannotate(exit.cause) })
    }
  })
  
  return {
    subscribe: store.subscribe
  }
}

// Refreshable effect store
export function refreshableEffectStore<A, E>(
  effectFn: () => Effect.Effect<A, E, never>,
  runtime: Runtime.Runtime<never> = Runtime.defaultRuntime
) {
  type State = 
    | { _tag: "loading" }
    | { _tag: "success"; data: A }
    | { _tag: "error"; error: E }
  
  const store = writable<State>({ _tag: "loading" })
  
  const load = () => {
    store.set({ _tag: "loading" })
    Effect.runPromiseExit(effectFn(), { runtime }).then(exit => {
      if (Exit.isSuccess(exit)) {
        store.set({ _tag: "success", data: exit.value })
      } else {
        store.set({ _tag: "error", error: Exit.unannotate(exit.cause) })
      }
    })
  }
  
  load() // Initial load
  
  return {
    subscribe: store.subscribe,
    refresh: load
  }
}
```

### 5.2 Using Effect in Svelte Components

```svelte
<!-- UserProfile.svelte -->
<script lang="ts">
  import { effectStore } from "$lib/effect-store"
  import { fetchUser } from "$lib/api"
  
  export let userId: string
  
  $: userStore = effectStore(fetchUser(userId))
</script>

{#if $userStore._tag === "loading"}
  <p>Loading...</p>
{:else if $userStore._tag === "error"}
  <p>Error: {JSON.stringify($userStore.error)}</p>
{:else}
  <div>
    <h1>{$userStore.data.name}</h1>
    <p>{$userStore.data.email}</p>
  </div>
{/if}
```

### 5.3 Derived Effect Stores

```typescript
import { derived } from "svelte/store"
import { Effect, pipe } from "effect"

// Base store
const userIdStore = writable("123")

// Derived effect store
const userStore = derived(
  userIdStore,
  ($userId, set) => {
    if (!$userId) {
      set({ _tag: "loading" })
      return
    }
    
    Effect.runPromiseExit(fetchUser($userId)).then(exit => {
      if (Exit.isSuccess(exit)) {
        set({ _tag: "success", data: exit.value })
      } else {
        set({ _tag: "error", error: Exit.unannotate(exit.cause) })
      }
    })
  },
  { _tag: "loading" } as State
)

// Multiple dependencies
const dashboardStore = derived(
  [userStore, postsStore, statsStore],
  ([$user, $posts, $stats]) => {
    if ($user._tag === "success" && 
        $posts._tag === "success" && 
        $stats._tag === "success") {
      return {
        _tag: "success" as const,
        data: {
          user: $user.data,
          posts: $posts.data,
          stats: $stats.data
        }
      }
    }
    return { _tag: "loading" as const }
  }
)
```

### 5.4 SvelteKit Server Integration

```typescript
// src/routes/api/users/[id]/+server.ts
import { json, error } from "@sveltejs/kit"
import { Effect, Exit } from "effect"
import type { RequestHandler } from "./$types"

export const GET: RequestHandler = async ({ params }) => {
  const result = await Effect.runPromiseExit(
    fetchUser(params.id)
  )
  
  if (Exit.isSuccess(result)) {
    return json(result.value)
  } else {
    const err = Exit.unannotate(result.cause)
    throw error(err._tag === "NotFound" ? 404 : 500, {
      message: String(err)
    })
  }
}

// src/routes/users/[id]/+page.ts
import { Effect } from "effect"
import type { PageLoad } from "./$types"

export const load: PageLoad = async ({ params, fetch }) => {
  const result = await Effect.runPromiseExit(
    Effect.tryPromise({
      try: () => fetch(`/api/users/${params.id}`).then(r => r.json()),
      catch: () => ({ _tag: "LoadFailed" as const })
    })
  )
  
  if (Exit.isSuccess(result)) {
    return { user: result.value }
  } else {
    throw new Error("Failed to load user")
  }
}
```

### 5.5 Reactive Form Validation

```svelte
<!-- SignupForm.svelte -->
<script lang="ts">
  import { writable, derived } from "svelte/store"
  import { Effect, Exit } from "effect"
  
  const email = writable("")
  const password = writable("")
  const username = writable("")
  
  // Validate on change
  const emailErrors = derived(email, $email => {
    if ($email === "") return []
    
    const result = Effect.runSyncExit(
      validateField($email, "email", required, emailValidator)
    )
    
    return Exit.isSuccess(result) ? [] : [Exit.unannotate(result.cause)]
  })
  
  const passwordErrors = derived(password, $password => {
    if ($password === "") return []
    
    const result = Effect.runSyncExit(
      validateField($password, "password", required, minLength(8))
    )
    
    return Exit.isSuccess(result) ? [] : [Exit.unannotate(result.cause)]
  })
  
  const isValid = derived(
    [emailErrors, passwordErrors],
    ([$emailErrors, $passwordErrors]) =>
      $emailErrors.length === 0 && $passwordErrors.length === 0
  )
  
  async function handleSubmit() {
    const form = {
      email: $email,
      password: $password,
      username: $username
    }
    
    const result = await Effect.runPromiseExit(
      validateSignupForm(form).pipe(
        Effect.flatMap(validForm => submitForm(validForm))
      )
    )
    
    if (Exit.isSuccess(result)) {
      console.log("Success!")
    }
  }
</script>

<form on:submit|preventDefault={handleSubmit}>
  <div>
    <input
      type="email"
      bind:value={$email}
      placeholder="Email"
    />
    {#each $emailErrors as error}
      <span class="error">{error._tag}</span>
    {/each}
  </div>
  
  <div>
    <input
      type="password"
      bind:value={$password}
      placeholder="Password"
    />
    {#each $passwordErrors as error}
      <span class="error">{error._tag}</span>
    {/each}
  </div>
  
  <button type="submit" disabled={!$isValid}>
    Sign Up
  </button>
</form>
```

---

## Module 6: Advanced Patterns

### 6.1 Resource Management

```typescript
import { Effect, pipe } from "effect"

// Acquire-use-release pattern
const withConnection = <A, E>(
  use: (conn: Connection) => Effect.Effect<A, E, never>
): Effect.Effect<A, E | ConnectionError> =>
  Effect.acquireUseRelease(
    // Acquire
    pipe(
      Effect.tryPromise({
        try: () => createConnection(),
        catch: () => ({ _tag: "ConnectionFailed" as const })
      }),
      Effect.tap(conn => Console.log("Connection acquired"))
    ),
    // Use
    use,
    // Release
    (conn, exit) =>
      pipe(
        Effect.sync(() => conn.close()),
        Effect.tap(() => Console.log("Connection closed")),
        Effect.catchAll(() => Effect.void)
      )
  )

// Usage
const queryDatabase = (sql: string) =>
  withConnection(conn =>
    Effect.tryPromise({
      try: () => conn.query(sql),
      catch: () => ({ _tag: "QueryFailed" as const })
    })
  )

// Scope for multiple resources
const withResources = Effect.gen(function* () {
  const db = yield* Effect.acquireRelease(
    openDatabase(),
    (db) => Effect.sync(() => db.close())
  )
  
  const cache = yield* Effect.acquireRelease(
    openCache(),
    (cache) => Effect.sync(() => cache.disconnect())
  )
  
  // Use both resources
  yield* performWork(db, cache)
})
```

### 6.2 Streaming and Chunking

```typescript
import { Effect, Stream, Chunk } from "effect"

// Process items in chunks
const processUsers = (userIds: string[]) =>
  pipe(
    Stream.fromIterable(userIds),
    Stream.mapEffect(id => fetchUser(id), { concurrency: 5 }),
    Stream.tap(user => Console.log(`Processed: ${user.name}`)),
    Stream.runCollect
  )

// Transform streams
const enrichedUsers = pipe(
  Stream.fromIterable(userIds),
  Stream.mapEffect(fetchUser),
  Stream.mapEffect(user =>
    pipe(
      fetchUserStats(user.id),
      Effect.map(stats => ({ ...user, stats }))
    )
  )
)

// Handle backpressure
const processWithBackpressure = pipe(
  Stream.fromIterable(items),
  Stream.buffer({ capacity: 100 }),
  Stream.mapEffect(processItem, { concurrency: 10 }),
  Stream.runDrain
)
```

### 6.3 Caching Strategies

```typescript
import { Effect, Cache, Duration } from "effect"

// Simple cache
const makeUserCache = Effect.gen(function* () {
  return yield* Cache.make({
    capacity: 100,
    timeToLive: Duration.minutes(5),
    lookup: (userId: string) => fetchUser(userId)
  })
})

// Use cache
const getUserWithCache = (userId: string) =>
  Effect.gen(function* () {
    const cache = yield* UserCacheService
    return yield* Cache.get(cache, userId)
  })

// Stale-while-revalidate
const staleWhileRevalidate = <A, E>(
  key: string,
  fetch: Effect.Effect<A, E>,
  cache: Cache.Cache<string, A>
) =>
  pipe(
    Cache.get(cache, key),
    Effect.race(
      pipe(
        fetch,
        Effect.tap(value => Cache.set(cache, key, value))
      )
    )
  )
```

### 6.4 Optimistic Updates

```typescript
import { Effect, Ref } from "effect"

interface TodosService {
  readonly todos: Ref.Ref<Todo[]>
  readonly addTodo: (text: string) => Effect.Effect<Todo, AddTodoError>
  readonly deleteTodo: (id: string) => Effect.Effect<void, DeleteTodoError>
}

const makeTodosService = Effect.gen(function* () {
  const todos = yield* Ref.make<Todo[]>([])
  const api = yield* ApiClient
  
  return {
    todos,
    
    addTodo: (text: string) => {
      const optimisticTodo: Todo = {
        id: `temp-${Date.now()}`,
        text,
        completed: false,
        _optimistic: true
      }
      
      return pipe(
        // Add optimistically
        Ref.update(todos, ts => [...ts, optimisticTodo]),
        Effect.flatMap(() =>
          api.post<Todo>("/todos", { text })
        ),
        Effect.tap(savedTodo =>
          // Replace optimistic with real
          Ref.update(todos, ts =>
            ts.map(t => t.id === optimisticTodo.id ? savedTodo : t)
          )
        ),
        Effect.tapError(() =>
          // Rollback on error
          Ref.update(todos, ts =>
            ts.filter(t => t.id !== optimisticTodo.id)
          )
        )
      )
    },
    
    deleteTodo: (id: string) =>
      pipe(
        // Find and remove optimistically
        Ref.get(todos),
        Effect.flatMap(ts => {
          const todo = ts.find(t => t.id === id)
          if (!todo) return Effect.fail({ _tag: "NotFound" as const })
          
          return pipe(
            Ref.update(todos, ts => ts.filter(t => t.id !== id)),
            Effect.flatMap(() => api.post("/todos/delete", { id })),
            Effect.tapError(() =>
              // Rollback on error
              Ref.update(todos, ts => [...ts, todo])
            )
          )
        })
      )
  }
})
```

### 6.5 Feature Flags Service

```typescript
import { Effect, Context, Layer, Ref } from "effect"

interface FeatureFlags {
  readonly isEnabled: (flag: string) => Effect.Effect<boolean>
  readonly getVariant: (flag: string) => Effect.Effect<string | null>
  readonly refresh: () => Effect.Effect<void>
}

const FeatureFlags = Context.GenericTag<FeatureFlags>("FeatureFlags")

type FlagConfig = Record<string, {
  enabled: boolean
  variant?: string
  rollout?: number // 0-100 percentage
}>

const makeFeatureFlags = Effect.gen(function* () {
  const api = yield* ApiClient
  const flags = yield* Ref.make<FlagConfig>({})
  
  const fetchFlags = pipe(
    api.get<FlagConfig>("/api/flags"),
    Effect.flatMap(config => Ref.set(flags, config))
  )
  
  // Initial fetch
  yield* fetchFlags
  
  const getUserBucket = (userId: string, flag: string): number => {
    // Consistent hashing for A/B testing
    let hash = 0
    const str = `${userId}-${flag}`
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i)
      hash = hash & hash
    }
    return Math.abs(hash) % 100
  }
  
  return FeatureFlags.of({
    isEnabled: (flag) =>
      pipe(
        Ref.get(flags),
        Effect.map(config => {
          const flagConfig = config[flag]
          if (!flagConfig) return false
          if (!flagConfig.enabled) return false
          
          if (flagConfig.rollout !== undefined) {
            // Get user from context if available
            const bucket = getUserBucket("default", flag)
            return bucket < flagConfig.rollout
          }
          
          return true
        })
      ),
    
    getVariant: (flag) =>
      pipe(
        Ref.get(flags),
        Effect.map(config => config[flag]?.variant ?? null)
      ),
    
    refresh: fetchFlags
  })
})

const FeatureFlagsLive = Layer.effect(FeatureFlags, makeFeatureFlags)

// Usage in components
const ConditionalFeature = () => {
  const enabled = useRx(
    Rx.make((get) =>
      pipe(
        FeatureFlags,
        Effect.flatMap(ff => ff.isEnabled("new-ui"))
      )
    )
  )
  
  if (enabled._tag === "success" && enabled.value) {
    return <NewUI />
  }
  
  return <OldUI />
}
```

---

## Module 7: Real-World Project

### Project: Production Admin Dashboard

Build a complete admin dashboard with:
- User management (CRUD operations)
- Real-time activity feed
- Data export functionality
- Feature flags management
- Analytics visualization

### 7.1 Project Structure

```
src/
├── domain/
│   ├── User.ts              # Domain models
│   ├── Activity.ts
│   └── Analytics.ts
├── services/
│   ├── ApiClient.ts         # HTTP client service
│   ├── UserRepository.ts    # User data access
│   ├── ActivityStream.ts    # Real-time activity
│   ├── FeatureFlags.ts      # Feature management
│   └── ExportService.ts     # Data export
├── effects/
│   ├── runtime.ts           # Effect runtime setup
│   └── layers.ts            # Service layers composition
├── components/
│   ├── UserList.tsx         # React components
│   ├── ActivityFeed.tsx
│   └── AnalyticsChart.tsx
└── App.tsx
```

### 7.2 Domain Models

```typescript
// domain/User.ts
import { Schema } from "@effect/schema"

export const User = Schema.Struct({
  id: Schema.String,
  email: Schema.String,
  name: Schema.String,
  role: Schema.Literal("admin", "user", "viewer"),
  createdAt: Schema.Date,
  lastActive: Schema.Date
})

export type User = Schema.Schema.Type<typeof User>

// domain/Activity.ts
export const Activity = Schema.Struct({
  id: Schema.String,
  userId: Schema.String,
  action: Schema.String,
  resource: Schema.String,
  timestamp: Schema.Date,
  metadata: Schema.Record(Schema.String, Schema.Unknown)
})

export type Activity = Schema.Schema.Type<typeof Activity>

// Validation
export const validateUser = (data: unknown): Effect.Effect<User, ParseError> =>
  Schema.decode(User)(data)
```

### 7.3 User Repository

```typescript
// services/UserRepository.ts
import { Effect, Context, pipe } from "effect"
import { ApiClient } from "./ApiClient"

export interface UserRepository {
  readonly listUsers: () => Effect.Effect<User[], RepositoryError>
  readonly getUser: (id: string) => Effect.Effect<User, RepositoryError>
  readonly createUser: (data: CreateUserData) => Effect.Effect<User, RepositoryError>
  readonly updateUser: (id: string, data: Partial<User>) => Effect.Effect<User, RepositoryError>
  readonly deleteUser: (id: string) => Effect.Effect<void, RepositoryError>
}

export const UserRepository = Context.GenericTag<UserRepository>("UserRepository")

type RepositoryError =
  | { _tag: "NotFound"; id: string }
  | { _tag: "ValidationError"; errors: unknown }
  | { _tag: "NetworkError"; cause: unknown }

export const makeUserRepository = Effect.gen(function* () {
  const api = yield* ApiClient
  
  return UserRepository.of({
    listUsers: () =>
      pipe(
        api.get<unknown[]>("/users"),
        Effect.flatMap(data =>
          Effect.all(data.map(validateUser), { concurrency: "unbounded" })
        ),
        Effect.mapError(error => ({
          _tag: "NetworkError" as const,
          cause: error
        }))
      ),
    
    getUser: (id) =>
      pipe(
        api.get<unknown>(`/users/${id}`),
        Effect.flatMap(validateUser),
        Effect.mapError(error =>
          error._tag === "InvalidResponse" && error.status === 404
            ? { _tag: "NotFound" as const, id }
            : { _tag: "NetworkError" as const, cause: error }
        )
      ),
    
    createUser: (data) =>
      pipe(
        api.post<unknown>("/users", data),
        Effect.flatMap(validateUser),
        Effect.mapError(error => ({
          _tag: "NetworkError" as const,
          cause: error
        }))
      ),
    
    updateUser: (id, data) =>
      pipe(
        api.post<unknown>(`/users/${id}`, data),
        Effect.flatMap(validateUser),
        Effect.mapError(error => ({
          _tag: "NetworkError" as const,
          cause: error
        }))
      ),
    
    deleteUser: (id) =>
      pipe(
        api.post("/users/delete", { id }),
        Effect.asVoid,
        Effect.mapError(error => ({
          _tag: "NetworkError" as const,
          cause: error
        }))
      )
  })
})

export const UserRepositoryLive = Layer.effect(
  UserRepository,
  makeUserRepository
)
```

### 7.4 Real-Time Activity Stream

```typescript
// services/ActivityStream.ts
import { Effect, Stream, Context, Schedule } from "effect"

export interface ActivityStream {
  readonly stream: Stream.Stream<Activity, StreamError>
}

export const ActivityStream = Context.GenericTag<ActivityStream>("ActivityStream")

type StreamError = { _tag: "StreamError"; cause: unknown }

export const makeActivityStream = Effect.gen(function* () {
  const api = yield* ApiClient
  
  // Poll for new activities
  const pollActivities = pipe(
    api.get<unknown[]>("/activities/recent"),
    Effect.flatMap(data =>
      Effect.all(data.map(validateActivity), { concurrency: "unbounded" })
    ),
    Effect.retry(Schedule.exponential("1 second")),
    Effect.mapError(cause => ({ _tag: "StreamError" as const, cause }))
  )
  
  return ActivityStream.of({
    stream: pipe(
      Stream.repeatEffect(pollActivities),
      Stream.flatMap(activities => Stream.fromIterable(activities)),
      Stream.schedule(Schedule.fixed("5 seconds"))
    )
  })
})

export const ActivityStreamLive = Layer.effect(
  ActivityStream,
  makeActivityStream
)
```

### 7.5 React Components with Effect-RX

```typescript
// components/UserList.tsx
import { Rx, useRx } from "@effect-rx/rx-react"
import { Effect, pipe } from "effect"
import { UserRepository } from "../services/UserRepository"

// Create reactive user list
const usersRx = Rx.family(() =>
  Rx.make((get) =>
    pipe(
      UserRepository,
      Effect.flatMap(repo => repo.listUsers())
    )
  )
)

// Refreshable version
const refreshableUsersRx = Rx.make((get) => {
  const trigger = get(refreshTrigger)
  return pipe(
    UserRepository,
    Effect.flatMap(repo => repo.listUsers()),
    Effect.tap(() => Console.log(`Users loaded (trigger: ${trigger})`))
  )
})

const refreshTrigger = Rx.make(0)

export function UserList() {
  const users = useRx(refreshableUsersRx)
  const [, setTrigger] = useRx(refreshTrigger)
  
  const refresh = () => setTrigger(prev => prev + 1)
  
  if (users._tag === "loading") {
    return <div>Loading users...</div>
  }
  
  if (users._tag === "error") {
    return (
      <div>
        <p>Error loading users</p>
        <button onClick={refresh}>Retry</button>
      </div>
    )
  }
  
  return (
    <div>
      <div className="header">
        <h2>Users</h2>
        <button onClick={refresh}>Refresh</button>
      </div>
      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Role</th>
            <th>Last Active</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {users.value.map(user => (
            <UserRow key={user.id} user={user} onUpdate={refresh} />
          ))}
        </tbody>
      </table>
    </div>
  )
}

function UserRow({ user, onUpdate }: { user: User; onUpdate: () => void }) {
  const [deleting, setDeleting] = useState(false)
  
  const handleDelete = async () => {
    setDeleting(true)
    
    const result = await Effect.runPromiseExit(
      pipe(
        UserRepository,
        Effect.flatMap(repo => repo.deleteUser(user.id))
      )
    )
    
    if (Exit.isSuccess(result)) {
      onUpdate()
    } else {
      alert("Failed to delete user")
      setDeleting(false)
    }
  }
  
  return (
    <tr>
      <td>{user.name}</td>
      <td>{user.email}</td>
      <td>{user.role}</td>
      <td>{user.lastActive.toLocaleDateString()}</td>
      <td>
        <button onClick={handleDelete} disabled={deleting}>
          {deleting ? "Deleting..." : "Delete"}
        </button>
      </td>
    </tr>
  )
}
```

### 7.6 Activity Feed Component

```typescript
// components/ActivityFeed.tsx
import { useEffect, useState } from "react"
import { Effect, Stream } from "effect"
import { ActivityStream } from "../services/ActivityStream"

export function ActivityFeed() {
  const [activities, setActivities] = useState<Activity[]>([])
  const [error, setError] = useState<string | null>(null)
  
  useEffect(() => {
    const program = pipe(
      ActivityStream,
      Effect.flatMap(stream =>
        Stream.runForEach(stream.stream, activity =>
          Effect.sync(() => {
            setActivities(prev => [activity, ...prev].slice(0, 50))
          })
        )
      ),
      Effect.catchAll(error =>
        Effect.sync(() => setError(String(error)))
      )
    )
    
    const fiber = Effect.runFork(program)
    
    return () => {
      Effect.runFork(fiber.pipe(Effect.flatMap(f => f.interrupt)))
    }
  }, [])
  
  if (error) {
    return <div>Error: {error}</div>
  }
  
  return (
    <div className="activity-feed">
      <h3>Recent Activity</h3>
      <ul>
        {activities.map(activity => (
          <li key={activity.id}>
            <span className="action">{activity.action}</span>
            <span className="resource">{activity.resource}</span>
            <span className="time">
              {activity.timestamp.toLocaleTimeString()}
            </span>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### 7.7 Export Service

```typescript
// services/ExportService.ts
import { Effect, Context, Stream, pipe } from "effect"

export interface ExportService {
  readonly exportUsers: (format: "csv" | "json") => Effect.Effect<Blob, ExportError>
  readonly exportActivities: (
    startDate: Date,
    endDate: Date
  ) => Effect.Effect<Blob, ExportError>
}

export const ExportService = Context.GenericTag<ExportService>("ExportService")

type ExportError = { _tag: "ExportFailed"; cause: unknown }

export const makeExportService = Effect.gen(function* () {
  const userRepo = yield* UserRepository
  
  const toCSV = (data: unknown[]): string => {
    if (data.length === 0) return ""
    
    const headers = Object.keys(data[0] as object)
    const rows = data.map(row =>
      headers.map(h => JSON.stringify((row as any)[h])).join(",")
    )
    
    return [headers.join(","), ...rows].join("\n")
  }
  
  return ExportService.of({
    exportUsers: (format) =>
      pipe(
        userRepo.listUsers(),
        Effect.map(users =>
          format === "csv"
            ? new Blob([toCSV(users)], { type: "text/csv" })
            : new Blob([JSON.stringify(users, null, 2)], { type: "application/json" })
        ),
        Effect.mapError(cause => ({ _tag: "ExportFailed" as const, cause }))
      ),
    
    exportActivities: (startDate, endDate) =>
      pipe(
        // Fetch activities in date range
        api.get<unknown[]>(
          `/activities?start=${startDate.toISOString()}&end=${endDate.toISOString()}`
        ),
        Effect.map(data => 
          new Blob([JSON.stringify(data, null, 2)], { type: "application/json" })
        ),
        Effect.mapError(cause => ({ _tag: "ExportFailed" as const, cause }))
      )
  })
})

export const ExportServiceLive = Layer.effect(ExportService, makeExportService)
```

### 7.8 Complete Layer Composition

```typescript
// effects/layers.ts
import { Layer } from "effect"
import { ApiClientLive } from "../services/ApiClient"
import { UserRepositoryLive } from "../services/UserRepository"
import { ActivityStreamLive } from "../services/ActivityStream"
import { FeatureFlagsLive } from "../services/FeatureFlags"
import { ExportServiceLive } from "../services/ExportService"

// Compose all layers
export const AppLayer = Layer.mergeAll(
  UserRepositoryLive,
  ActivityStreamLive,
  FeatureFlagsLive,
  ExportServiceLive
).pipe(
  Layer.provide(ApiClientLive)
)

// Runtime with all services
export const runtime = Layer.toRuntime(AppLayer).pipe(
  Effect.runSync
)
```

---

## Module 8: Testing Strategies

### 8.1 Testing Pure Effects

```typescript
import { Effect, Exit } from "effect"
import { describe, it, expect } from "vitest"

describe("User validation", () => {
  it("should validate correct user data", async () => {
    const validUser = {
      id: "123",
      email: "test@example.com",
      name: "Test User",
      role: "user" as const,
      createdAt: new Date(),
      lastActive: new Date()
    }
    
    const result = await Effect.runPromiseExit(validateUser(validUser))
    
    expect(Exit.isSuccess(result)).toBe(true)
    if (Exit.isSuccess(result)) {
      expect(result.value).toEqual(validUser)
    }
  })
  
  it("should fail on invalid email", async () => {
    const invalidUser = {
      id: "123",
      email: "not-an-email",
      name: "Test User",
      role: "user" as const
    }
    
    const result = await Effect.runPromiseExit(validateUser(invalidUser))
    
    expect(Exit.isFailure(result)).toBe(true)
  })
})
```

### 8.2 Testing with Mocks

```typescript
import { Effect, Context, Layer } from "effect"

// Test doubles
const MockUserRepository: UserRepository = {
  listUsers: () => Effect.succeed([
    {
      id: "1",
      email: "user1@test.com",
      name: "User 1",
      role: "user",
      createdAt: new Date(),
      lastActive: new Date()
    }
  ]),
  
  getUser: (id) =>
    id === "1"
      ? Effect.succeed({ id: "1", name: "User 1", /* ... */ })
      : Effect.fail({ _tag: "NotFound", id }),
  
  createUser: (data) =>
    Effect.succeed({ id: "new", ...data, createdAt: new Date(), lastActive: new Date() }),
  
  updateUser: (id, data) =>
    Effect.succeed({ id, ...data as User }),
  
  deleteUser: (id) => Effect.void
}

const MockUserRepositoryLayer = Layer.succeed(
  UserRepository,
  MockUserRepository
)

// Test with mocks
describe("User operations", () => {
  it("should list users", async () => {
    const program = pipe(
      UserRepository,
      Effect.flatMap(repo => repo.listUsers()),
      Effect.provide(MockUserRepositoryLayer)
    )
    
    const result = await Effect.runPromise(program)
    
    expect(result).toHaveLength(1)
    expect(result[0].name).toBe("User 1")
  })
})
```

### 8.3 Property-Based Testing

```typescript
import { Effect } from "effect"
import { describe, it } from "vitest"
import * as fc from "fast-check"

describe("User repository properties", () => {
  it("should roundtrip user data", async () => {
    await fc.assert(
      fc.asyncProperty(
        fc.record({
          id: fc.string(),
          email: fc.emailAddress(),
          name: fc.string({ minLength: 1 }),
          role: fc.constantFrom("admin", "user", "viewer")
        }),
        async (userData) => {
          const program = pipe(
            UserRepository,
            Effect.flatMap(repo => repo.createUser(userData)),
            Effect.flatMap(created => repo.getUser(created.id)),
            Effect.provide(MockUserRepositoryLayer)
          )
          
          const result = await Effect.runPromise(program)
          
          expect(result.email).toBe(userData.email)
          expect(result.name).toBe(userData.name)
        }
      )
    )
  })
})
```

---

## Module 9: Performance and Optimization

### 9.1 Batching Requests

```typescript
import { Effect, RequestResolver, Request } from "effect"

// Define request types
interface GetUser extends Request.Request<User, UserNotFound> {
  readonly _tag: "GetUser"
  readonly id: string
}

const GetUser = Request.tagged<GetUser>("GetUser")

// Batch resolver
const UserResolver = RequestResolver.makeBatched((requests: GetUser[]) => {
  const ids = requests.map(r => r.id)
  
  return pipe(
    api.post<User[]>("/users/batch", { ids }),
    Effect.flatMap(users => {
      const userMap = new Map(users.map(u => [u.id, u]))
      
      return Effect.forEach(requests, request => {
        const user = userMap.get(request.id)
        return user
          ? Request.succeed(request, user)
          : Request.fail(request, { _tag: "UserNotFound", id: request.id })
      })
    })
  )
})

// Usage
const getUser = (id: string) =>
  Effect.request(GetUser({ id }), UserResolver)

// These will be automatically batched
const users = Effect.all([
  getUser("1"),
  getUser("2"),
  getUser("3")
], { batching: true })
```

### 9.2 Memoization

```typescript
import { Effect, Cache, Duration } from "effect"

// Memoize expensive computations
const makeAnalyticsCache = Effect.gen(function* () {
  return yield* Cache.make({
    capacity: 50,
    timeToLive: Duration.minutes(15),
    lookup: (dateRange: string) =>
      pipe(
        fetchAnalytics(dateRange),
        Effect.tap(() => Console.log(`Computing analytics for ${dateRange}`))
      )
  })
})

const getAnalytics = (dateRange: string) =>
  Effect.gen(function* () {
    const cache = yield* AnalyticsCache
    return yield* Cache.get(cache, dateRange)
  })
```

### 9.3 Lazy Loading

```typescript
import { Rx } from "@effect-rx/rx-react"
import { Effect } from "effect"

// Load data only when needed
const lazyUserDetails = (userId: string) =>
  Rx.make((get) => {
    const shouldLoad = get(userDetailsVisibleRx(userId))
    
    if (!shouldLoad) {
      return Effect.succeed(null)
    }
    
    return fetchUserDetails(userId)
  })

function UserCard({ userId }: { userId: string }) {
  const [expanded, setExpanded] = useState(false)
  const details = useRx(lazyUserDetails(userId))
  
  return (
    <div>
      <button onClick={() => setExpanded(!expanded)}>
        {expanded ? "Hide" : "Show"} Details
      </button>
      
      {expanded && details._tag === "success" && details.value && (
        <UserDetails data={details.value} />
      )}
    </div>
  )
}
```

---

## Module 10: Best Practices

### 10.1 Error Handling Guidelines

```typescript
// 1. Use discriminated unions for errors
type DomainError =
  | { _tag: "NotFound"; id: string }
  | { _tag: "ValidationError"; field: string; message: string }
  | { _tag: "Unauthorized" }
  | { _tag: "RateLimited"; retryAfter: number }

// 2. Handle errors at appropriate boundaries
const userWorkflow = pipe(
  fetchUser(id),
  Effect.flatMap(validateUser),
  Effect.flatMap(enrichUser),
  // Handle specific errors
  Effect.catchTag("NotFound", () => createDefaultUser()),
  Effect.catchTag("ValidationError", error =>
    Effect.fail({ _tag: "WorkflowFailed", cause: error })
  ),
  // Log unexpected errors
  Effect.tapError(error =>
    Console.error("Unexpected error in user workflow:", error)
  )
)

// 3. Provide user-friendly error messages at UI boundary
const handleError = (error: DomainError): string => {
  switch (error._tag) {
    case "NotFound":
      return "User not found"
    case "ValidationError":
      return `Invalid ${error.field}: ${error.message}`
    case "Unauthorized":
      return "You don't have permission"
    case "RateLimited":
      return `Too many requests. Try again in ${error.retryAfter} seconds`
  }
}
```

### 10.2 Service Design Patterns

```typescript
// 1. Keep services focused and composable
interface UserService {
  readonly getUser: (id: string) => Effect.Effect<User, UserError>
  readonly listUsers: () => Effect.Effect<User[], UserError>
}

interface AuthService {
  readonly checkPermission: (
    userId: string,
    permission: string
  ) => Effect.Effect<boolean, AuthError>
}

// 2. Compose services for complex operations
const authorizedUserFetch = (userId: string) =>
  Effect.gen(function* () {
    const auth = yield* AuthService
    const users = yield* UserService
    
    const canView = yield* auth.checkPermission(currentUserId, "view:users")
    
    if (!canView) {
      return yield* Effect.fail({ _tag: "Unauthorized" as const })
    }
    
    return yield* users.getUser(userId)
  })

// 3. Use layers for dependency management
const AppLayer = Layer.mergeAll(
  UserServiceLive,
  AuthServiceLive
).pipe(
  Layer.provide(ApiClientLive),
  Layer.provide(ConfigLive)
)
```

### 10.3 React/Svelte Integration Patterns

```typescript
// 1. Colocate effects with components
function UserProfile({ userId }: { userId: string }) {
  // Define effect near usage
  const userEffect = useMemo(
    () =>
      pipe(
        UserService,
        Effect.flatMap(s => s.getUser(userId))
      ),
    [userId]
  )
  
  const user = useRx(Rx.make(() => userEffect))
  
  // ...
}

// 2. Separate business logic from UI
const useUserData = (userId: string) => {
  const user = useRx(Rx.make(() =>
    pipe(
      UserService,
      Effect.flatMap(s => s.getUser(userId))
    )
  ))
  
  const posts = useRx(Rx.make(() =>
    pipe(
      PostService,
      Effect.flatMap(s => s.getUserPosts(userId))
    )
  ))
  
  return { user, posts }
}

// 3. Handle loading states consistently
type LoadingState<T, E> =
  | { _tag: "loading" }
  | { _tag: "success"; data: T }
  | { _tag: "error"; error: E }

const renderLoadingState = <T, E>(
  state: LoadingState<T, E>,
  render: (data: T) => ReactNode
): ReactNode => {
  switch (state._tag) {
    case "loading":
      return <Spinner />
    case "error":
      return <ErrorMessage error={state.error} />
    case "success":
      return render(state.data)
  }
}
```

### 10.4 Type Safety Tips

```typescript
// 1. Use branded types for IDs
type UserId = string & { readonly _brand: "UserId" }
type PostId = string & { readonly _brand: "PostId" }

const UserId = (id: string): UserId => id as UserId
const PostId = (id: string): PostId => id as PostId

// This prevents mixing IDs
const getUser = (id: UserId) => { /* ... */ }
const getPost = (id: PostId) => { /* ... */ }

getUser(UserId("123")) // ✓
getUser(PostId("123")) // ✗ Type error

// 2. Exhaustive pattern matching with never
const handleError = (error: DomainError): string => {
  switch (error._tag) {
    case "NotFound":
      return "Not found"
    case "ValidationError":
      return "Invalid"
    default:
      const _exhaustive: never = error
      return _exhaustive
  }
}

// 3. Use Schema for runtime validation
import { Schema } from "@effect/schema"

const ApiResponse = Schema.Struct({
  data: Schema.Array(User),
  total: Schema.Number,
  page: Schema.Number
})

const parseResponse = (data: unknown) =>
  Schema.decode(ApiResponse)(data)
```

---

## Appendix A: Effect-TS Cheat Sheet

### Core Functions

```typescript
// Creating Effects
Effect.succeed(value)              // Pure success
Effect.fail(error)                 // Pure failure
Effect.sync(() => value)           // Lazy computation
Effect.promise(() => promise)      // From promise
Effect.try({ try, catch })         // Catching exceptions

// Transforming
Effect.map(f)                      // Transform success
Effect.flatMap(f)                  // Chain effects
Effect.tap(f)                      // Side effect
Effect.mapError(f)                 // Transform error

// Error Handling
Effect.catchAll(handler)           // Catch all errors
Effect.catchTag(tag, handler)      // Catch specific error
Effect.retry(policy)               // Retry on failure
Effect.option                      // Error to Option

// Combining
Effect.all([e1, e2, ...])         // Sequence/parallel
Effect.race(e1, e2)               // First to complete
Effect.zip(e1, e2)                // Combine two

// Running
Effect.runPromise(effect)          // Run async
Effect.runSync(effect)             // Run sync
Effect.runPromiseExit(effect)      // With exit info

// Context
Context.GenericTag<T>(name)        // Create tag
Effect.provideService(tag, impl)   // Provide service
Effect.provide(layer)              // Provide layer
```

### Common Patterns

```typescript
// Pipeline pattern
pipe(
  initialValue,
  step1,
  step2,
  step3
)

// Generator pattern
Effect.gen(function* () {
  const a = yield* effect1
  const b = yield* effect2
  return combine(a, b)
})

// Error recovery
pipe(
  riskyEffect,
  Effect.catchTag("SpecificError", fallback),
  Effect.tapError(logError)
)

// Conditional logic
Effect.if(condition, {
  onTrue: () => effect1,
  onFalse: () => effect2
})
```

---

## Appendix B: Resources

### Official Documentation
- Effect Website: https://effect.website
- API Reference: https://effect-ts.github.io/effect
- Discord Community: https://discord.gg/effect-ts

### Libraries
- `effect`: Core Effect library
- `@effect/platform`: Platform utilities (HTTP, File system)
- `@effect/schema`: Runtime type validation
- `@effect-rx/rx-react`: React integration
- `@effect/cli`: Command-line applications

### Learning Resources
- Effect Days (Conference talks)
- Effect YouTube Channel
- Community examples on GitHub

---

## Course Completion Project Ideas

1. **E-commerce Cart System**
   - Shopping cart with optimistic updates
   - Payment processing with retries
   - Inventory management with validation

2. **Collaborative Editor**
   - Real-time updates via WebSocket
   - Conflict resolution
   - Undo/redo with Effect

3. **Analytics Dashboard**
   - Data aggregation pipelines
   - Export to multiple formats
   - Real-time chart updates

4. **Content Management System**
   - CRUD operations with validation
   - Image upload and processing
   - Role-based permissions

5. **Chat Application**
   - Message streaming
   - Typing indicators
   - File sharing with progress

---

## Conclusion

Effect-TS brings the power of functional effect systems to TypeScript, enabling you to build robust, composable, and type-safe frontend applications. By mastering Effect's patterns for error handling, dependency injection, and composition, you can eliminate entire classes of bugs and create maintainable codebases that scale.

The key principles to remember:
- Effects are descriptions, not executions
- Compose effects using `pipe` and combinators
- Use the Context system for dependency injection
- Handle errors with typed discriminated unions
- Integrate with React/Svelte using Effect-RX
- Test with mocks and property-based testing

Continue exploring the Effect ecosystem, join the community, and apply these patterns to your production applications!
