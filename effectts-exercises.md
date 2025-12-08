# EffectTS Frontend Mastery - Exercises

**Practice Problems and Solutions**

This document contains hands-on exercises for each module of the EffectTS course. Each exercise includes:
- Problem statement
- Starter code
- Hints
- Complete solution
- Variations to explore

---

## Table of Contents

1. [Module 1: Effect Fundamentals](#module-1-exercises-effect-fundamentals)
2. [Module 2: Composition and Combinators](#module-2-exercises-composition-and-combinators)
3. [Module 3: Context and Dependency Injection](#module-3-exercises-context-and-dependency-injection)
4. [Module 4: React Integration](#module-4-exercises-react-integration)
5. [Module 5: Svelte Integration](#module-5-exercises-svelte-integration)
6. [Module 6: Advanced Patterns](#module-6-exercises-advanced-patterns)
7. [Bonus Exercises](#bonus-exercises)
8. [Testing Your Solutions](#testing-your-solutions)

---

## Module 1 Exercises: Effect Fundamentals

### Exercise 1.1: Basic Effect Construction

**Problem:** Create effects for common operations and understand their types.

```typescript
import { Effect } from "effect"

// TODO: Create an effect that succeeds with your name
const getName = // Your code here

// TODO: Create an effect that fails with "Invalid input"
const failEffect = // Your code here

// TODO: Create an effect that computes the square of a number
const square = (n: number) => // Your code here

// TODO: Create an effect from a promise that fetches data
const fetchData = (url: string) => // Your code here
```

**Hints:**
- Use `Effect.succeed()` for pure values
- Use `Effect.fail()` for failures
- Use `Effect.sync()` for lazy computations
- Use `Effect.promise()` for async operations

**Solution:**

```typescript
import { Effect } from "effect"

const getName = Effect.succeed("Karsten")

const failEffect = Effect.fail("Invalid input")

const square = (n: number) => Effect.sync(() => n * n)

const fetchData = (url: string) =>
  Effect.promise(() => fetch(url).then(res => res.json()))

// Test the effects
Effect.runPromise(getName).then(console.log) // "Karsten"
Effect.runPromise(square(5)).then(console.log) // 25
```

### Exercise 1.2: Safe Division

**Problem:** Implement a safe division function that handles division by zero.

```typescript
import { Effect } from "effect"

type MathError = { _tag: "DivisionByZero" }

const safeDivide = (a: number, b: number): Effect.Effect<number, MathError> => {
  // TODO: Implement safe division
}

// Test cases
await Effect.runPromise(safeDivide(10, 2)) // Should succeed with 5
await Effect.runPromise(safeDivide(10, 0)) // Should fail with DivisionByZero
```

**Solution:**

```typescript
import { Effect } from "effect"

type MathError = { _tag: "DivisionByZero" }

const safeDivide = (a: number, b: number): Effect.Effect<number, MathError> =>
  b === 0
    ? Effect.fail({ _tag: "DivisionByZero" })
    : Effect.succeed(a / b)

// Test with error handling
const result = await Effect.runPromise(
  safeDivide(10, 0).pipe(
    Effect.catchTag("DivisionByZero", () => Effect.succeed(0))
  )
)
console.log(result) // 0
```

**Variations:**
1. Add more math operations (sqrt with negative check, log with zero check)
2. Create a calculator that chains multiple operations
3. Add custom error messages with the operands

### Exercise 1.3: Error Recovery

**Problem:** Fetch user data with multiple fallback strategies.

```typescript
import { Effect, pipe } from "effect"

type FetchError = 
  | { _tag: "NetworkError" }
  | { _tag: "NotFound" }
  | { _tag: "Timeout" }

const fetchUser = (id: string): Effect.Effect<User, FetchError> => {
  // Simulated API call
  return Effect.fail({ _tag: "NotFound" })
}

// TODO: Implement fetchWithFallbacks that:
// 1. Tries to fetch from primary API
// 2. On NotFound, tries backup API
// 3. On NetworkError, retries 3 times
// 4. On Timeout, returns a default user
const fetchWithFallbacks = (id: string) => {
  // Your code here
}
```

**Solution:**

```typescript
import { Effect, pipe, Schedule } from "effect"

type FetchError = 
  | { _tag: "NetworkError" }
  | { _tag: "NotFound" }
  | { _tag: "Timeout" }

interface User {
  id: string
  name: string
  email: string
}

const fetchPrimaryAPI = (id: string): Effect.Effect<User, FetchError> =>
  Effect.fail({ _tag: "NotFound" })

const fetchBackupAPI = (id: string): Effect.Effect<User, FetchError> =>
  Effect.succeed({ id, name: "Backup User", email: "backup@example.com" })

const defaultUser: User = {
  id: "default",
  name: "Guest",
  email: "guest@example.com"
}

const fetchWithFallbacks = (id: string): Effect.Effect<User, never> =>
  pipe(
    fetchPrimaryAPI(id),
    Effect.retry({
      while: (error) => error._tag === "NetworkError",
      times: 3
    }),
    Effect.catchTag("NotFound", () => fetchBackupAPI(id)),
    Effect.catchTag("Timeout", () => Effect.succeed(defaultUser)),
    Effect.catchAll(() => Effect.succeed(defaultUser))
  )
```

---

## Module 2 Exercises: Composition and Combinators

### Exercise 2.1: Pipeline Transformation

**Problem:** Build a data transformation pipeline for processing user input.

```typescript
import { Effect, pipe } from "effect"

interface RawInput {
  text: string
  priority: string // "low" | "medium" | "high"
}

interface ProcessedInput {
  content: string
  priorityLevel: number
  wordCount: number
  timestamp: Date
}

// TODO: Implement these transformations
const trim = (input: RawInput): Effect.Effect<RawInput, never> => {
  // Trim whitespace from text
}

const validateNotEmpty = (input: RawInput): Effect.Effect<RawInput, ValidationError> => {
  // Fail if text is empty after trimming
}

const convertPriority = (input: RawInput): Effect.Effect<number, never> => {
  // Convert priority string to number (low=1, medium=2, high=3)
}

const countWords = (text: string): Effect.Effect<number, never> => {
  // Count words in text
}

// TODO: Compose these into a complete pipeline
const processInput = (input: RawInput): Effect.Effect<ProcessedInput, ValidationError> => {
  // Your code here
}
```

**Solution:**

```typescript
import { Effect, pipe } from "effect"

interface RawInput {
  text: string
  priority: string
}

interface ProcessedInput {
  content: string
  priorityLevel: number
  wordCount: number
  timestamp: Date
}

type ValidationError = { _tag: "EmptyInput" }

const trim = (input: RawInput): Effect.Effect<RawInput, never> =>
  Effect.succeed({
    ...input,
    text: input.text.trim()
  })

const validateNotEmpty = (input: RawInput): Effect.Effect<RawInput, ValidationError> =>
  input.text.length === 0
    ? Effect.fail({ _tag: "EmptyInput" })
    : Effect.succeed(input)

const convertPriority = (priority: string): number => {
  const map: Record<string, number> = { low: 1, medium: 2, high: 3 }
  return map[priority.toLowerCase()] ?? 1
}

const countWords = (text: string): number =>
  text.split(/\s+/).filter(word => word.length > 0).length

const processInput = (input: RawInput): Effect.Effect<ProcessedInput, ValidationError> =>
  pipe(
    trim(input),
    Effect.flatMap(validateNotEmpty),
    Effect.map(validated => ({
      content: validated.text,
      priorityLevel: convertPriority(validated.priority),
      wordCount: countWords(validated.text),
      timestamp: new Date()
    }))
  )

// Test
const testInput: RawInput = {
  text: "  Hello world  ",
  priority: "high"
}

Effect.runPromise(processInput(testInput)).then(console.log)
```

### Exercise 2.2: Parallel Data Fetching

**Problem:** Fetch data from multiple sources concurrently and combine results.

```typescript
import { Effect } from "effect"

interface UserProfile {
  user: User
  posts: Post[]
  comments: Comment[]
  followers: number
}

// TODO: Implement fetchUserProfile that fetches all data concurrently
const fetchUserProfile = (userId: string): Effect.Effect<UserProfile, FetchError> => {
  // Your code here
}

// Bonus: Add timeout of 5 seconds
// Bonus: Add retry logic for failed requests
```

**Solution:**

```typescript
import { Effect, pipe } from "effect"

interface User {
  id: string
  name: string
}

interface Post {
  id: string
  title: string
}

interface Comment {
  id: string
  text: string
}

interface UserProfile {
  user: User
  posts: Post[]
  comments: Comment[]
  followers: number
}

type FetchError = { _tag: "FetchFailed"; cause: unknown }

// Mock API calls
const fetchUser = (id: string): Effect.Effect<User, FetchError> =>
  Effect.succeed({ id, name: "Test User" })

const fetchUserPosts = (id: string): Effect.Effect<Post[], FetchError> =>
  Effect.succeed([{ id: "1", title: "First Post" }])

const fetchUserComments = (id: string): Effect.Effect<Comment[], FetchError> =>
  Effect.succeed([{ id: "1", text: "Great!" }])

const fetchFollowerCount = (id: string): Effect.Effect<number, FetchError> =>
  Effect.succeed(150)

const fetchUserProfile = (userId: string): Effect.Effect<UserProfile, FetchError> =>
  pipe(
    Effect.all(
      {
        user: fetchUser(userId),
        posts: fetchUserPosts(userId),
        comments: fetchUserComments(userId),
        followers: fetchFollowerCount(userId)
      },
      { concurrency: "unbounded" }
    ),
    Effect.timeout("5 seconds"),
    Effect.catchTag("TimeoutException", () =>
      Effect.fail({ _tag: "FetchFailed", cause: "Timeout" })
    ),
    Effect.retry({ times: 2 })
  )

// Test
Effect.runPromise(fetchUserProfile("123")).then(console.log)
```

### Exercise 2.3: Conditional Workflows

**Problem:** Implement a workflow that branches based on user permissions.

```typescript
import { Effect } from "effect"

interface User {
  id: string
  role: "admin" | "user" | "guest"
  permissions: string[]
}

interface Document {
  id: string
  content: string
  ownerId: string
}

type AccessError = 
  | { _tag: "Unauthorized" }
  | { _tag: "Forbidden" }

// TODO: Implement accessDocument that:
// 1. Checks if user is authenticated (not guest)
// 2. Checks if user has "read:documents" permission OR is document owner
// 3. Returns document if authorized
const accessDocument = (
  user: User,
  docId: string
): Effect.Effect<Document, AccessError> => {
  // Your code here
}
```

**Solution:**

```typescript
import { Effect, pipe } from "effect"

interface User {
  id: string
  role: "admin" | "user" | "guest"
  permissions: string[]
}

interface Document {
  id: string
  content: string
  ownerId: string
}

type AccessError = 
  | { _tag: "Unauthorized" }
  | { _tag: "Forbidden" }

const fetchDocument = (id: string): Effect.Effect<Document, never> =>
  Effect.succeed({
    id,
    content: "Secret document",
    ownerId: "owner123"
  })

const hasPermission = (user: User, permission: string): boolean =>
  user.permissions.includes(permission)

const isOwner = (user: User, doc: Document): boolean =>
  user.id === doc.ownerId

const accessDocument = (
  user: User,
  docId: string
): Effect.Effect<Document, AccessError> =>
  pipe(
    Effect.if(user.role === "guest", {
      onTrue: () => Effect.fail({ _tag: "Unauthorized" as const }),
      onFalse: () => fetchDocument(docId)
    }),
    Effect.flatMap(doc =>
      hasPermission(user, "read:documents") || isOwner(user, doc)
        ? Effect.succeed(doc)
        : Effect.fail({ _tag: "Forbidden" as const })
    )
  )

// Test cases
const adminUser: User = {
  id: "admin1",
  role: "admin",
  permissions: ["read:documents", "write:documents"]
}

const guestUser: User = {
  id: "guest1",
  role: "guest",
  permissions: []
}

Effect.runPromise(accessDocument(adminUser, "doc1")).then(console.log)
Effect.runPromise(accessDocument(guestUser, "doc1")).catch(console.error)
```

---

## Module 3 Exercises: Context and Dependency Injection

### Exercise 3.1: Building a Logger Service

**Problem:** Create a configurable logging service with different implementations.

```typescript
import { Effect, Context, Layer } from "effect"

// TODO: Define Logger service interface
interface Logger {
  readonly debug: (message: string) => Effect.Effect<void>
  readonly info: (message: string) => Effect.Effect<void>
  readonly warn: (message: string) => Effect.Effect<void>
  readonly error: (message: string) => Effect.Effect<void>
}

// TODO: Create Logger tag

// TODO: Implement ConsoleLogger

// TODO: Implement FileLogger (simulated)

// TODO: Create layers for both implementations

// TODO: Use logger in an application
const application = pipe(
  Logger,
  Effect.flatMap(logger =>
    pipe(
      logger.info("Application started"),
      Effect.flatMap(() => logger.debug("Processing data...")),
      Effect.flatMap(() => logger.warn("Low memory warning"))
    )
  )
)
```

**Solution:**

```typescript
import { Effect, Context, Layer, Console } from "effect"

interface Logger {
  readonly debug: (message: string) => Effect.Effect<void>
  readonly info: (message: string) => Effect.Effect<void>
  readonly warn: (message: string) => Effect.Effect<void>
  readonly error: (message: string) => Effect.Effect<void>
}

class Logger extends Context.Tag("Logger")<Logger, Logger>() {}

// Console implementation
const ConsoleLogger: Logger = {
  debug: (message) => Console.log(`[DEBUG] ${message}`),
  info: (message) => Console.log(`[INFO] ${message}`),
  warn: (message) => Console.warn(`[WARN] ${message}`),
  error: (message) => Console.error(`[ERROR] ${message}`)
}

const ConsoleLoggerLive = Layer.succeed(Logger, ConsoleLogger)

// File implementation (simulated)
const FileLogger: Logger = {
  debug: (message) =>
    Effect.sync(() => {
      // In real implementation: fs.appendFile('app.log', message)
      console.log(`[FILE:DEBUG] ${message}`)
    }),
  info: (message) =>
    Effect.sync(() => console.log(`[FILE:INFO] ${message}`)),
  warn: (message) =>
    Effect.sync(() => console.log(`[FILE:WARN] ${message}`)),
  error: (message) =>
    Effect.sync(() => console.log(`[FILE:ERROR] ${message}`))
}

const FileLoggerLive = Layer.succeed(Logger, FileLogger)

// Application using logger
const application = pipe(
  Logger,
  Effect.flatMap(logger =>
    pipe(
      logger.info("Application started"),
      Effect.flatMap(() => logger.debug("Processing data...")),
      Effect.flatMap(() => logger.warn("Low memory warning")),
      Effect.map(() => "Complete")
    )
  )
)

// Run with different logger implementations
Effect.runPromise(application.pipe(Effect.provide(ConsoleLoggerLive)))
Effect.runPromise(application.pipe(Effect.provide(FileLoggerLive)))
```

### Exercise 3.2: Service Composition

**Problem:** Build a UserService that depends on Logger and ApiClient services.

```typescript
import { Effect, Context, Layer } from "effect"

// Given services
interface Logger {
  readonly log: (message: string) => Effect.Effect<void>
}

interface ApiClient {
  readonly get: <T>(url: string) => Effect.Effect<T, ApiError>
  readonly post: <T, B>(url: string, body: B) => Effect.Effect<T, ApiError>
}

// TODO: Define UserService that uses both Logger and ApiClient
interface UserService {
  readonly getUser: (id: string) => Effect.Effect<User, UserError>
  readonly createUser: (data: CreateUserData) => Effect.Effect<User, UserError>
}

// TODO: Create UserService tag

// TODO: Implement makeUserService

// TODO: Create UserServiceLive layer

// TODO: Compose all layers
```

**Solution:**

```typescript
import { Effect, Context, Layer, Console } from "effect"

interface User {
  id: string
  name: string
  email: string
}

type CreateUserData = Pick<User, "name" | "email">

type ApiError = { _tag: "NetworkError"; cause: unknown }
type UserError = { _tag: "UserNotFound" } | { _tag: "CreateFailed" }

interface Logger {
  readonly log: (message: string) => Effect.Effect<void>
}

class Logger extends Context.Tag("Logger")<Logger, Logger>() {}

interface ApiClient {
  readonly get: <T>(url: string) => Effect.Effect<T, ApiError>
  readonly post: <T, B>(url: string, body: B) => Effect.Effect<T, ApiError>
}

class ApiClient extends Context.Tag("ApiClient")<ApiClient, ApiClient>() {}

interface UserService {
  readonly getUser: (id: string) => Effect.Effect<User, UserError>
  readonly createUser: (data: CreateUserData) => Effect.Effect<User, UserError>
}

class UserService extends Context.Tag("UserService")<UserService, UserService>() {}

const UserServiceLive = Layer.effect(UserService, pipe(
  Logger,
  Effect.flatMap(logger =>
    pipe(
      ApiClient,
      Effect.map(api => ({
        getUser: (id) =>
          pipe(
            logger.log(`Fetching user ${id}`),
            Effect.flatMap(() => api.get<User>(`/users/${id}`)),
            Effect.mapError(() => ({ _tag: "UserNotFound" as const })),
            Effect.tap(user => logger.log(`Retrieved user ${user.name}`))
          ),

        createUser: (data) =>
          pipe(
            logger.log(`Creating user ${data.email}`),
            Effect.flatMap(() => api.post<User>("/users", data)),
            Effect.mapError(() => ({ _tag: "CreateFailed" as const })),
            Effect.tap(user => logger.log(`Created user ${user.id}`))
          )
      }))
    )
  )
))

// Mock implementations
const LoggerLive = Layer.succeed(Logger, {
  log: (message) => Console.log(message)
})

const ApiClientLive = Layer.succeed(ApiClient, {
  get: <T>(url: string) =>
    Effect.succeed({ id: "123", name: "Test", email: "test@example.com" } as T),
  post: <T, B>(url: string, body: B) =>
    Effect.succeed({ id: "new", ...body } as T)
})

// Compose all layers
const AppLayer = UserServiceLive.pipe(
  Layer.provide(Layer.merge(LoggerLive, ApiClientLive))
)

// Usage
const program = pipe(
  UserService,
  Effect.flatMap(userService => userService.getUser("123"))
)

Effect.runPromise(program.pipe(Effect.provide(AppLayer)))
```

### Exercise 3.3: Testing with Mocks

**Problem:** Create mock services for testing business logic.

```typescript
import { Effect, Context, Layer } from "effect"

// Given: A service that depends on external APIs
interface PaymentService {
  readonly processPayment: (
    amount: number,
    cardToken: string
  ) => Effect.Effect<PaymentResult, PaymentError>
}

// TODO: Create a mock PaymentService for testing
// TODO: Test a checkout workflow that uses PaymentService
// TODO: Verify the mock was called correctly
```

**Solution:**

```typescript
import { Effect, Context, Layer, Ref } from "effect"

interface PaymentResult {
  transactionId: string
  status: "success" | "failed"
  amount: number
}

type PaymentError = 
  | { _tag: "InsufficientFunds" }
  | { _tag: "InvalidCard" }
  | { _tag: "ProcessingError" }

interface PaymentService {
  readonly processPayment: (
    amount: number,
    cardToken: string
  ) => Effect.Effect<PaymentResult, PaymentError>
}

class PaymentService extends Context.Tag("PaymentService")<PaymentService, PaymentService>() {}

// Mock implementation that tracks calls
interface MockPaymentService extends PaymentService {
  readonly getCalls: () => Effect.Effect<Array<{ amount: number; cardToken: string }>>
}

const MockPaymentServiceLayer = Layer.effect(PaymentService, pipe(
  Ref.make<Array<{ amount: number; cardToken: string }>>([]),
  Effect.map(calls => ({
    processPayment: (amount: number, cardToken: string) =>
      pipe(
        Ref.update(calls, prev => [...prev, { amount, cardToken }]),
        Effect.flatMap(() =>
          cardToken === "invalid"
            ? Effect.fail({ _tag: "InvalidCard" as const })
            : Effect.succeed({
                transactionId: `txn-${Date.now()}`,
                status: "success" as const,
                amount
              })
        )
      ),

    getCalls: () => Ref.get(calls)
  }))
))

// Business logic to test
const checkout = (amount: number, cardToken: string) =>
  pipe(
    PaymentService,
    Effect.flatMap(payment => payment.processPayment(amount, cardToken))
  )

// Test
const test = pipe(
  checkout(100, "valid-token"),
  Effect.tap(result => Effect.sync(() => console.log("Payment result:", result))),
  Effect.map(result => result.status === "success")
)

Effect.runPromise(test.pipe(Effect.provide(MockPaymentServiceLayer)))
```

---

## Module 4 Exercises: React Integration

### Exercise 4.1: User Profile Component

**Problem:** Build a React component that fetches and displays user data using Effect.

```typescript
import { Effect } from "effect"
import { Rx, useRx } from "@effect-rx/rx-react"

// TODO: Create a user profile component that:
// 1. Fetches user data on mount
// 2. Shows loading state
// 3. Handles errors with retry button
// 4. Displays user information when loaded

interface User {
  id: string
  name: string
  email: string
  avatar: string
}

const fetchUser = (id: string): Effect.Effect<User, FetchError> => {
  // Simulated API call
}

function UserProfile({ userId }: { userId: string }) {
  // Your code here
}
```

**Solution:**

```typescript
import { Effect, pipe } from "effect"
import { Rx, useRx } from "@effect-rx/rx-react"
import { useState } from "react"

interface User {
  id: string
  name: string
  email: string
  avatar: string
}

type FetchError = { _tag: "NetworkError" } | { _tag: "NotFound" }

const fetchUser = (id: string): Effect.Effect<User, FetchError> =>
  Effect.promise(() =>
    fetch(`/api/users/${id}`)
      .then(res => res.json())
      .catch(() => ({ _tag: "NetworkError" as const }))
  )

// Create reactive user data
const userRx = (userId: string) =>
  Rx.make(() => fetchUser(userId))

function UserProfile({ userId }: { userId: string }) {
  const user = useRx(userRx(userId))
  const [retryCount, setRetryCount] = useState(0)
  
  // Force retry by creating new Rx
  const retry = () => setRetryCount(prev => prev + 1)
  
  if (user._tag === "loading") {
    return (
      <div className="user-profile loading">
        <div className="spinner" />
        <p>Loading user...</p>
      </div>
    )
  }
  
  if (user._tag === "error") {
    return (
      <div className="user-profile error">
        <p>Failed to load user profile</p>
        <button onClick={retry}>Retry</button>
      </div>
    )
  }
  
  return (
    <div className="user-profile">
      <img src={user.value.avatar} alt={user.value.name} />
      <h2>{user.value.name}</h2>
      <p>{user.value.email}</p>
      <button onClick={retry}>Refresh</button>
    </div>
  )
}

export default UserProfile
```

### Exercise 4.2: Form with Validation

**Problem:** Create a signup form with Effect-based validation.

```typescript
import { Effect } from "effect"

// TODO: Implement validators
// TODO: Create form component with real-time validation
// TODO: Show field-specific errors
// TODO: Disable submit when invalid
// TODO: Handle submission with Effect

type ValidationError =
  | { _tag: "Required"; field: string }
  | { _tag: "InvalidEmail"; field: string }
  | { _tag: "TooShort"; field: string; min: number }

interface SignupForm {
  email: string
  password: string
  confirmPassword: string
}
```

**Solution:**

```typescript
import { Effect, pipe } from "effect"
import { useState, useEffect } from "react"

type ValidationError =
  | { _tag: "Required"; field: string }
  | { _tag: "InvalidEmail"; field: string }
  | { _tag: "TooShort"; field: string; min: number }
  | { _tag: "PasswordMismatch" }

interface SignupForm {
  email: string
  password: string
  confirmPassword: string
}

// Validators
const required = (value: string, field: string): Effect.Effect<string, ValidationError> =>
  value.trim().length === 0
    ? Effect.fail({ _tag: "Required", field })
    : Effect.succeed(value)

const email = (value: string, field: string): Effect.Effect<string, ValidationError> =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)
    ? Effect.succeed(value)
    : Effect.fail({ _tag: "InvalidEmail", field })

const minLength = (min: number) => 
  (value: string, field: string): Effect.Effect<string, ValidationError> =>
    value.length < min
      ? Effect.fail({ _tag: "TooShort", field, min })
      : Effect.succeed(value)

const validateField = (
  value: string,
  field: string,
  validators: Array<(v: string, f: string) => Effect.Effect<string, ValidationError>>
): Effect.Effect<string, ValidationError[]> =>
  pipe(
    Effect.all(
      validators.map(v => v(value, field)),
      { concurrency: "unbounded" }
    ),
    Effect.map(() => value),
    Effect.mapError(errors => Array.isArray(errors) ? errors : [errors])
  )

const validateForm = (form: SignupForm): Effect.Effect<SignupForm, ValidationError[]> =>
  pipe(
    Effect.all({
      email: validateField(form.email, "email", [required, email]),
      password: validateField(form.password, "password", [required, minLength(8)]),
      confirmPassword: required(form.confirmPassword, "confirmPassword")
    }),
    Effect.flatMap(() =>
      form.password !== form.confirmPassword
        ? Effect.fail([{ _tag: "PasswordMismatch" as const }])
        : Effect.succeed(form)
    ),
    Effect.mapError(errors => Array.isArray(errors) ? errors : [errors])
  )

function SignupForm() {
  const [form, setForm] = useState<SignupForm>({
    email: "",
    password: "",
    confirmPassword: ""
  })
  
  const [errors, setErrors] = useState<ValidationError[]>([])
  const [touched, setTouched] = useState<Record<string, boolean>>({})
  const [isValid, setIsValid] = useState(false)
  
  // Validate on change
  useEffect(() => {
    const result = Effect.runSyncExit(validateForm(form))
    
    if (result._tag === "Success") {
      setErrors([])
      setIsValid(true)
    } else {
      const validationErrors = result.cause
      setErrors(Array.isArray(validationErrors) ? validationErrors : [validationErrors])
      setIsValid(false)
    }
  }, [form])
  
  const getFieldErrors = (field: string) =>
    errors.filter(e => "field" in e && e.field === field)
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    
    // Mark all fields as touched
    setTouched({ email: true, password: true, confirmPassword: true })
    
    const result = await Effect.runPromiseExit(
      validateForm(form).pipe(
        Effect.flatMap(validForm =>
          Effect.promise(() =>
            fetch("/api/signup", {
              method: "POST",
              headers: { "Content-Type": "application/json" },
              body: JSON.stringify(validForm)
            })
          )
        )
      )
    )
    
    if (result._tag === "Success") {
      console.log("Signup successful!")
    } else {
      console.error("Signup failed:", result.cause)
    }
  }
  
  const renderFieldErrors = (field: string) => {
    if (!touched[field]) return null
    
    const fieldErrors = getFieldErrors(field)
    if (fieldErrors.length === 0) return null
    
    return (
      <div className="field-errors">
        {fieldErrors.map((error, i) => (
          <span key={i} className="error">
            {error._tag === "Required" && "This field is required"}
            {error._tag === "InvalidEmail" && "Invalid email address"}
            {error._tag === "TooShort" && `Minimum ${error.min} characters`}
          </span>
        ))}
      </div>
    )
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <div className="form-field">
        <label>Email</label>
        <input
          type="email"
          value={form.email}
          onChange={e => setForm({ ...form, email: e.target.value })}
          onBlur={() => setTouched({ ...touched, email: true })}
        />
        {renderFieldErrors("email")}
      </div>
      
      <div className="form-field">
        <label>Password</label>
        <input
          type="password"
          value={form.password}
          onChange={e => setForm({ ...form, password: e.target.value })}
          onBlur={() => setTouched({ ...touched, password: true })}
        />
        {renderFieldErrors("password")}
      </div>
      
      <div className="form-field">
        <label>Confirm Password</label>
        <input
          type="password"
          value={form.confirmPassword}
          onChange={e => setForm({ ...form, confirmPassword: e.target.value })}
          onBlur={() => setTouched({ ...touched, confirmPassword: true })}
        />
        {renderFieldErrors("confirmPassword")}
        {touched.confirmPassword && errors.some(e => e._tag === "PasswordMismatch") && (
          <span className="error">Passwords do not match</span>
        )}
      </div>
      
      <button type="submit" disabled={!isValid}>
        Sign Up
      </button>
    </form>
  )
}

export default SignupForm
```

### Exercise 4.3: Infinite Scroll List

**Problem:** Implement an infinite scroll list with Effect for data fetching.

```typescript
// TODO: Create a list component that:
// 1. Loads initial page of items
// 2. Detects when user scrolls near bottom
// 3. Loads next page automatically
// 4. Shows loading indicator at bottom
// 5. Handles end of data

interface ListItem {
  id: string
  title: string
  description: string
}

const fetchPage = (page: number): Effect.Effect<ListItem[], FetchError> => {
  // Your implementation
}

function InfiniteList() {
  // Your code here
}
```

**Solution:**

```typescript
import { Effect, pipe } from "effect"
import { useState, useEffect, useRef } from "react"

interface ListItem {
  id: string
  title: string
  description: string
}

type FetchError = { _tag: "NetworkError" }

const fetchPage = (page: number, pageSize: number = 20): Effect.Effect<ListItem[], FetchError> =>
  Effect.promise(() =>
    fetch(`/api/items?page=${page}&size=${pageSize}`)
      .then(res => res.json())
      .catch(() => ({ _tag: "NetworkError" as const }))
  )

function InfiniteList() {
  const [items, setItems] = useState<ListItem[]>([])
  const [page, setPage] = useState(0)
  const [loading, setLoading] = useState(false)
  const [hasMore, setHasMore] = useState(true)
  const [error, setError] = useState<FetchError | null>(null)
  
  const observerTarget = useRef<HTMLDivElement>(null)
  
  const loadMore = async () => {
    if (loading || !hasMore) return
    
    setLoading(true)
    setError(null)
    
    const result = await Effect.runPromiseExit(fetchPage(page))
    
    if (result._tag === "Success") {
      const newItems = result.value
      
      if (newItems.length === 0) {
        setHasMore(false)
      } else {
        setItems(prev => [...prev, ...newItems])
        setPage(prev => prev + 1)
      }
    } else {
      setError(result.cause)
    }
    
    setLoading(false)
  }
  
  // Load initial page
  useEffect(() => {
    loadMore()
  }, [])
  
  // Intersection observer for infinite scroll
  useEffect(() => {
    const observer = new IntersectionObserver(
      entries => {
        if (entries[0].isIntersecting && !loading && hasMore) {
          loadMore()
        }
      },
      { threshold: 1.0 }
    )
    
    const currentTarget = observerTarget.current
    if (currentTarget) {
      observer.observe(currentTarget)
    }
    
    return () => {
      if (currentTarget) {
        observer.unobserve(currentTarget)
      }
    }
  }, [loading, hasMore])
  
  return (
    <div className="infinite-list">
      {items.map(item => (
        <div key={item.id} className="list-item">
          <h3>{item.title}</h3>
          <p>{item.description}</p>
        </div>
      ))}
      
      {loading && (
        <div className="loading-indicator">
          <div className="spinner" />
          <p>Loading more...</p>
        </div>
      )}
      
      {error && (
        <div className="error-message">
          <p>Failed to load more items</p>
          <button onClick={loadMore}>Retry</button>
        </div>
      )}
      
      {!hasMore && items.length > 0 && (
        <div className="end-message">
          <p>No more items to load</p>
        </div>
      )}
      
      <div ref={observerTarget} style={{ height: "20px" }} />
    </div>
  )
}

export default InfiniteList
```

---

## Module 5 Exercises: Svelte Integration

### Exercise 5.1: Svelte Counter with Effect

**Problem:** Create a Svelte counter with Effect-based persistence.

```svelte
<!-- TODO: Create a counter that:
1. Increments/decrements
2. Saves to localStorage using Effect
3. Loads from localStorage on mount
4. Handles save errors gracefully
-->

<script lang="ts">
  import { Effect } from "effect"
  
  // Your code here
</script>

<div>
  <button>-</button>
  <span>{count}</span>
  <button>+</button>
</div>
```

**Solution:**

```svelte
<script lang="ts">
  import { Effect, pipe } from "effect"
  import { writable } from "svelte/store"
  import { onMount } from "svelte"
  
  type StorageError = { _tag: "StorageError"; cause: unknown }
  
  const STORAGE_KEY = "counter-value"
  
  const saveToStorage = (value: number): Effect.Effect<void, StorageError> =>
    Effect.try({
      try: () => localStorage.setItem(STORAGE_KEY, String(value)),
      catch: (cause) => ({ _tag: "StorageError" as const, cause })
    })
  
  const loadFromStorage = (): Effect.Effect<number, StorageError> =>
    Effect.try({
      try: () => {
        const stored = localStorage.getItem(STORAGE_KEY)
        return stored ? parseInt(stored, 10) : 0
      },
      catch: (cause) => ({ _tag: "StorageError" as const, cause })
    })
  
  const count = writable(0)
  
  onMount(() => {
    Effect.runPromise(
      loadFromStorage().pipe(
        Effect.catchAll(() => Effect.succeed(0))
      )
    ).then(value => count.set(value))
  })
  
  const increment = () => {
    count.update(n => {
      const newValue = n + 1
      Effect.runPromise(saveToStorage(newValue))
      return newValue
    })
  }
  
  const decrement = () => {
    count.update(n => {
      const newValue = n - 1
      Effect.runPromise(saveToStorage(newValue))
      return newValue
    })
  }
</script>

<div class="counter">
  <button on:click={decrement}>-</button>
  <span class="count">{$count}</span>
  <button on:click={increment}>+</button>
</div>

<style>
  .counter {
    display: flex;
    gap: 1rem;
    align-items: center;
  }
  
  .count {
    font-size: 2rem;
    font-weight: bold;
    min-width: 3rem;
    text-align: center;
  }
  
  button {
    font-size: 1.5rem;
    padding: 0.5rem 1rem;
  }
</style>
```

### Exercise 5.2: Todo List with Effect

**Problem:** Build a complete todo list application.

```svelte
<!-- TODO: Create a todo list that:
1. Adds new todos
2. Toggles completion
3. Deletes todos
4. Persists to API using Effect
5. Shows loading states
6. Handles errors
-->

<script lang="ts">
  import { Effect } from "effect"
  
  interface Todo {
    id: string
    text: string
    completed: boolean
  }
  
  // Your code here
</script>

<!-- Your template here -->
```

**Solution:**

```svelte
<script lang="ts">
  import { Effect, pipe } from "effect"
  import { writable, derived } from "svelte/store"
  import { onMount } from "svelte"
  
  interface Todo {
    id: string
    text: string
    completed: boolean
  }
  
  type TodoError = 
    | { _tag: "FetchFailed" }
    | { _tag: "SaveFailed" }
    | { _tag: "DeleteFailed" }
  
  // API functions
  const fetchTodos = (): Effect.Effect<Todo[], TodoError> =>
    Effect.tryPromise({
      try: () => fetch("/api/todos").then(r => r.json()),
      catch: () => ({ _tag: "FetchFailed" as const })
    })
  
  const createTodo = (text: string): Effect.Effect<Todo, TodoError> =>
    Effect.tryPromise({
      try: () =>
        fetch("/api/todos", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ text, completed: false })
        }).then(r => r.json()),
      catch: () => ({ _tag: "SaveFailed" as const })
    })
  
  const toggleTodo = (todo: Todo): Effect.Effect<Todo, TodoError> =>
    Effect.tryPromise({
      try: () =>
        fetch(`/api/todos/${todo.id}`, {
          method: "PATCH",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ completed: !todo.completed })
        }).then(r => r.json()),
      catch: () => ({ _tag: "SaveFailed" as const })
    })
  
  const deleteTodo = (id: string): Effect.Effect<void, TodoError> =>
    Effect.tryPromise({
      try: () => fetch(`/api/todos/${id}`, { method: "DELETE" }).then(() => {}),
      catch: () => ({ _tag: "DeleteFailed" as const })
    })
  
  // State
  const todos = writable<Todo[]>([])
  const newTodoText = writable("")
  const loading = writable(false)
  const error = writable<string | null>(null)
  
  // Derived stores
  const activeTodos = derived(todos, $todos =>
    $todos.filter(t => !t.completed)
  )
  
  const completedTodos = derived(todos, $todos =>
    $todos.filter(t => t.completed)
  )
  
  // Load todos on mount
  onMount(() => {
    loading.set(true)
    Effect.runPromise(
      fetchTodos().pipe(
        Effect.tap(loadedTodos => Effect.sync(() => todos.set(loadedTodos))),
        Effect.catchAll(err => {
          error.set("Failed to load todos")
          return Effect.void
        }),
        Effect.tap(() => Effect.sync(() => loading.set(false)))
      )
    )
  })
  
  // Actions
  const addTodo = async () => {
    const text = $newTodoText.trim()
    if (!text) return
    
    loading.set(true)
    error.set(null)
    
    await Effect.runPromise(
      createTodo(text).pipe(
        Effect.tap(newTodo =>
          Effect.sync(() => {
            todos.update(t => [...t, newTodo])
            newTodoText.set("")
          })
        ),
        Effect.catchAll(() => {
          error.set("Failed to create todo")
          return Effect.void
        }),
        Effect.tap(() => Effect.sync(() => loading.set(false)))
      )
    )
  }
  
  const toggleComplete = async (todo: Todo) => {
    await Effect.runPromise(
      toggleTodo(todo).pipe(
        Effect.tap(updated =>
          Effect.sync(() =>
            todos.update(t =>
              t.map(item => (item.id === updated.id ? updated : item))
            )
          )
        ),
        Effect.catchAll(() => {
          error.set("Failed to update todo")
          return Effect.void
        })
      )
    )
  }
  
  const removeTodo = async (id: string) => {
    await Effect.runPromise(
      deleteTodo(id).pipe(
        Effect.tap(() =>
          Effect.sync(() =>
            todos.update(t => t.filter(item => item.id !== id))
          )
        ),
        Effect.catchAll(() => {
          error.set("Failed to delete todo")
          return Effect.void
        })
      )
    )
  }
</script>

<div class="todo-app">
  <h1>Todo List</h1>
  
  {#if $error}
    <div class="error">{$error}</div>
  {/if}
  
  <div class="input-section">
    <input
      type="text"
      bind:value={$newTodoText}
      on:keypress={(e) => e.key === 'Enter' && addTodo()}
      placeholder="What needs to be done?"
      disabled={$loading}
    />
    <button on:click={addTodo} disabled={$loading || !$newTodoText.trim()}>
      Add
    </button>
  </div>
  
  {#if $loading && $todos.length === 0}
    <div class="loading">Loading todos...</div>
  {:else}
    <div class="todos-section">
      <h2>Active ({$activeTodos.length})</h2>
      <ul>
        {#each $activeTodos as todo (todo.id)}
          <li>
            <input
              type="checkbox"
              checked={todo.completed}
              on:change={() => toggleComplete(todo)}
            />
            <span>{todo.text}</span>
            <button on:click={() => removeTodo(todo.id)}>Delete</button>
          </li>
        {/each}
      </ul>
      
      {#if $completedTodos.length > 0}
        <h2>Completed ({$completedTodos.length})</h2>
        <ul>
          {#each $completedTodos as todo (todo.id)}
            <li class="completed">
              <input
                type="checkbox"
                checked={todo.completed}
                on:change={() => toggleComplete(todo)}
              />
              <span>{todo.text}</span>
              <button on:click={() => removeTodo(todo.id)}>Delete</button>
            </li>
          {/each}
        </ul>
      {/if}
    </div>
  {/if}
</div>

<style>
  .todo-app {
    max-width: 600px;
    margin: 0 auto;
    padding: 2rem;
  }
  
  .error {
    background: #fee;
    color: #c00;
    padding: 1rem;
    border-radius: 4px;
    margin-bottom: 1rem;
  }
  
  .input-section {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 2rem;
  }
  
  .input-section input {
    flex: 1;
    padding: 0.5rem;
    font-size: 1rem;
  }
  
  .loading {
    text-align: center;
    padding: 2rem;
    color: #666;
  }
  
  ul {
    list-style: none;
    padding: 0;
  }
  
  li {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem;
    border-bottom: 1px solid #eee;
  }
  
  li span {
    flex: 1;
  }
  
  li.completed span {
    text-decoration: line-through;
    color: #999;
  }
</style>
```

---

## Module 6 Exercises: Advanced Patterns

### Exercise 6.1: Request Batching

**Problem:** Implement request batching for user data fetches.

```typescript
import { Effect, RequestResolver, Request } from "effect"

// TODO: Define GetUser request
// TODO: Create batched resolver
// TODO: Use in application to demonstrate batching

interface User {
  id: string
  name: string
}

// Simulated batch API
const fetchUsersBatch = (ids: string[]): Promise<User[]> => {
  console.log(`Fetching batch: ${ids.join(", ")}`)
  return Promise.resolve(ids.map(id => ({ id, name: `User ${id}` })))
}
```

**Solution:**

```typescript
import { Effect, Request, RequestResolver, pipe } from "effect"

interface User {
  id: string
  name: string
}

type UserNotFound = { _tag: "UserNotFound"; id: string }

// Define request type
interface GetUser extends Request.Request<User, UserNotFound> {
  readonly _tag: "GetUser"
  readonly id: string
}

const GetUser = Request.tagged<GetUser>("GetUser")

// Simulated batch API
const fetchUsersBatch = (ids: string[]): Promise<User[]> => {
  console.log(`Fetching batch: ${ids.join(", ")}`)
  return Promise.resolve(ids.map(id => ({ id, name: `User ${id}` })))
}

// Create batched resolver
const UserResolver = RequestResolver.makeBatched(
  (requests: readonly GetUser[]) => {
    const ids = requests.map(r => r.id)
    
    return pipe(
      Effect.promise(() => fetchUsersBatch(ids)),
      Effect.flatMap(users => {
        const userMap = new Map(users.map(u => [u.id, u]))
        
        return Effect.forEach(requests, request => {
          const user = userMap.get(request.id)
          return user
            ? Request.succeed(request, user)
            : Request.fail(request, { _tag: "UserNotFound" as const, id: request.id })
        })
      })
    )
  }
)

// Helper function to get user
const getUser = (id: string) =>
  Effect.request(GetUser({ id, _tag: "GetUser" }), UserResolver)

// Usage - these will be batched into a single request
const program = pipe(
  Effect.all(
    [
      getUser("1"),
      getUser("2"),
      getUser("3")
    ],
    { batching: true }
  ),
  Effect.tap(users => Effect.sync(() => console.log("Users:", users)))
)

// Run the program
Effect.runPromise(program)
// Output: Fetching batch: 1, 2, 3
```

### Exercise 6.2: Caching with TTL

**Problem:** Implement a caching layer with time-to-live.

```typescript
import { Effect, Cache, Duration } from "effect"

// TODO: Create a cache for expensive computations
// TODO: Add TTL of 5 minutes
// TODO: Demonstrate cache hits and misses

const expensiveComputation = (input: string): Effect.Effect<string, never> => {
  console.log(`Computing for: ${input}`)
  return Effect.sync(() => input.toUpperCase())
}
```

**Solution:**

```typescript
import { Effect, Cache, Duration, pipe } from "effect"

const expensiveComputation = (input: string): Effect.Effect<string, never> => {
  console.log(`Computing for: ${input}`)
  return pipe(
    Effect.sleep(Duration.seconds(1)), // Simulate expensive operation
    Effect.flatMap(() => Effect.sync(() => input.toUpperCase()))
  )
}

// Create cache
const makeComputationCache = Cache.make({
  capacity: 100,
  timeToLive: Duration.minutes(5),
  lookup: (key: string) => expensiveComputation(key)
})

// Usage
const program = pipe(
  makeComputationCache,
  Effect.flatMap(cache =>
    pipe(
      Effect.sync(() => console.log("First call:")),
      Effect.flatMap(() => Cache.get(cache, "hello")),
      Effect.tap(result1 => Effect.sync(() => console.log("Result:", result1))),
      Effect.flatMap(() => Effect.sync(() => console.log("\nSecond call (cached):"))),
      Effect.flatMap(() => Cache.get(cache, "hello")),
      Effect.tap(result2 => Effect.sync(() => console.log("Result:", result2))),
      Effect.flatMap(() => Effect.sync(() => console.log("\nDifferent key:"))),
      Effect.flatMap(() => Cache.get(cache, "world")),
      Effect.tap(result3 => Effect.sync(() => console.log("Result:", result3))),
      Effect.flatMap(() => Effect.sync(() => console.log("\nWait for expiration..."))),
      Effect.flatMap(() => Effect.sleep(Duration.minutes(6))),
      Effect.flatMap(() => Effect.sync(() => console.log("\nAfter expiration:"))),
      Effect.flatMap(() => Cache.get(cache, "hello")),
      Effect.tap(result4 => Effect.sync(() => console.log("Result:", result4)))
    )
  )
)

Effect.runPromise(program)
```

### Exercise 6.3: Optimistic UI Updates

**Problem:** Implement optimistic updates for a todo list.

```typescript
import { Effect, Ref } from "effect"

// TODO: Create a todo service with optimistic updates
// TODO: On add: show immediately, then confirm with server
// TODO: On delete: remove immediately, rollback on error
// TODO: Show visual indication for optimistic items

interface Todo {
  id: string
  text: string
  completed: boolean
  _optimistic?: boolean
}
```

**Solution:**

```typescript
import { Effect, Ref, pipe } from "effect"

interface Todo {
  id: string
  text: string
  completed: boolean
  _optimistic?: boolean
}

type TodoError = { _tag: "SaveFailed" } | { _tag: "DeleteFailed" }

// Simulated API
const apiCreateTodo = (text: string): Effect.Effect<Todo, TodoError> =>
  pipe(
    Effect.sleep("500 millis"),
    Effect.flatMap(() =>
      Effect.succeed({
        id: `server-${Date.now()}`,
        text,
        completed: false
      })
    )
  )

const apiDeleteTodo = (id: string): Effect.Effect<void, TodoError> =>
  pipe(
    Effect.sleep("500 millis"),
    Effect.flatMap(() => Effect.void)
  )

// Todo service with optimistic updates
interface TodoService {
  readonly todos: Ref.Ref<Todo[]>
  readonly addTodo: (text: string) => Effect.Effect<Todo, TodoError>
  readonly deleteTodo: (id: string) => Effect.Effect<void, TodoError>
}

const makeTodoService = pipe(
  Ref.make<Todo[]>([]),
  Effect.map(todos => ({
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
        Effect.flatMap(() => apiCreateTodo(text)),
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
        Ref.get(todos),
        Effect.flatMap(ts => {
          const todo = ts.find(t => t.id === id)
          if (!todo) {
            return Effect.fail({ _tag: "DeleteFailed" as const })
          }

          return pipe(
            // Remove optimistically
            Ref.update(todos, ts => ts.filter(t => t.id !== id)),
            Effect.flatMap(() => apiDeleteTodo(id)),
            Effect.tapError(() =>
              // Rollback on error
              Ref.update(todos, ts => [...ts, todo])
            )
          )
        })
      )
  }))
)

// Usage example
const demo = pipe(
  makeTodoService,
  Effect.flatMap(service =>
    pipe(
      Effect.sync(() => console.log("Adding todo...")),
      Effect.flatMap(() => service.addTodo("Buy milk")),
      Effect.flatMap(() => Ref.get(service.todos)),
      Effect.tap(todos1 => Effect.sync(() => console.log("Todos (with optimistic):", todos1))),
      Effect.flatMap(() => Effect.sleep("1 second")),
      Effect.flatMap(() => Ref.get(service.todos)),
      Effect.tap(todos2 => Effect.sync(() => console.log("Todos (after server):", todos2))),
      Effect.flatMap(todos2 =>
        pipe(
          Effect.sync(() => console.log("\nDeleting todo...")),
          Effect.flatMap(() =>
            todos2[0]
              ? service.deleteTodo(todos2[0].id)
              : Effect.void
          ),
          Effect.flatMap(() => Ref.get(service.todos)),
          Effect.tap(todos3 => Effect.sync(() => console.log("Todos (after delete):", todos3)))
        )
      )
    )
  )
)

Effect.runPromise(demo)
```

---

## Bonus Exercises

### Bonus 1: Real-time Notifications

Build a notification system that streams updates in real-time.

### Bonus 2: Feature Flag A/B Testing

Implement a feature flag system with user bucketing for A/B tests.

### Bonus 3: Undo/Redo System

Create an undo/redo system using Effect and Ref for state management.

### Bonus 4: Offline-First Sync

Build a system that queues operations when offline and syncs when reconnected.

---

## Testing Your Solutions

All exercises include starter code and solutions. To test your solutions:

1. Set up a test project with Effect-TS
2. Copy the exercise code
3. Implement your solution
4. Compare with the provided solution
5. Try the suggested variations

Remember: The goal is to understand the patterns, not just copy solutions. Try to solve each exercise before looking at the solution!
