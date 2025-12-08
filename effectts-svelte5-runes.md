# Effect-TS with Svelte 5 Runes

**Modern Svelte Integration using Runes and SvelteKit**

This guide shows how to integrate Effect-TS with Svelte 5's new runes system (`$state`, `$derived`, `$effect`) and SvelteKit's latest patterns.

---

## Table of Contents

1. [Runes Overview](#runes-overview)
2. [Effect with $state](#effect-with-state)
3. [Effect with $derived](#effect-with-derived)
4. [Effect with $effect](#effect-with-effect)
5. [SvelteKit Integration](#sveltekit-integration)
6. [Complete Examples](#complete-examples)
7. [Best Practices](#best-practices)

---

## Runes Overview

Svelte 5 introduces runes for reactivity:

```typescript
// $state - reactive state
let count = $state(0)

// $derived - computed values
let doubled = $derived(count * 2)

// $effect - side effects
$effect(() => {
  console.log(`Count is now ${count}`)
})

// $props - component props
let { userId } = $props<{ userId: string }>()
```

---

[↑ Back to Top](#table-of-contents)

## Effect with $state

### Basic Pattern

```typescript
<script lang="ts">
  import { Effect } from "effect"
  import { onMount } from "svelte"
  
  type LoadingState<T, E> =
    | { status: "loading" }
    | { status: "success"; data: T }
    | { status: "error"; error: E }
  
  let { userId } = $props<{ userId: string }>()
  
  let userState = $state<LoadingState<User, UserError>>({
    status: "loading"
  })
  
  onMount(() => {
    Effect.runPromise(fetchUser(userId))
      .then(data => {
        userState = { status: "success", data }
      })
      .catch(error => {
        userState = { status: "error", error }
      })
  })
</script>

{#if userState.status === "loading"}
  <p>Loading...</p>
{:else if userState.status === "error"}
  <p>Error: {userState.error._tag}</p>
{:else}
  <div>
    <h1>{userState.data.name}</h1>
    <p>{userState.data.email}</p>
  </div>
{/if}
```

### Reactive Fetching

React to prop changes automatically:

```typescript
<script lang="ts">
  import { Effect } from "effect"
  
  let { userId } = $props<{ userId: string }>()
  
  let userState = $state<LoadingState<User, UserError>>({
    status: "loading"
  })
  
  // Automatically refetch when userId changes
  $effect(() => {
    userState = { status: "loading" }
    
    Effect.runPromise(fetchUser(userId))
      .then(data => {
        userState = { status: "success", data }
      })
      .catch(error => {
        userState = { status: "error", error }
      })
  })
</script>
```

### Custom Rune Helper

Create a reusable `$effectState` helper:

```typescript
// lib/effect-runes.svelte.ts
import { Effect, Exit } from "effect"

export type AsyncState<T, E> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: E }

export function createEffectState<A, E>(
  effect: Effect.Effect<A, E, never>
) {
  let state = $state<AsyncState<A, E>>({ status: "loading" })
  
  Effect.runPromiseExit(effect).then(exit => {
    if (Exit.isSuccess(exit)) {
      state = { status: "success", data: exit.value }
    } else {
      state = { status: "error", error: Exit.unannotate(exit.cause) }
    }
  })
  
  return {
    get current() {
      return state
    }
  }
}

// Usage in component
<script lang="ts">
  import { createEffectState } from "$lib/effect-runes.svelte"
  
  let { userId } = $props<{ userId: string }>()
  
  let user = createEffectState(fetchUser(userId))
</script>

{#if user.current.status === "success"}
  <p>{user.current.data.name}</p>
{/if}
```

---

[↑ Back to Top](#table-of-contents)

## Effect with $derived

### Computed Effect Results

```typescript
<script lang="ts">
  import { Effect } from "effect"
  
  let searchQuery = $state("")
  
  // Derived computation with Effect
  let searchResults = $derived.by(() => {
    if (searchQuery.length < 3) {
      return { status: "idle" } as const
    }
    
    // This creates the effect but doesn't run it
    return {
      status: "pending" as const,
      effect: searchUsers(searchQuery)
    }
  })
  
  // Trigger effect when derived changes
  $effect(() => {
    if (searchResults.status === "pending") {
      Effect.runPromise(searchResults.effect)
        .then(data => {
          // Update state
        })
    }
  })
</script>
```

### Better Pattern: Separate State and Derivation

```typescript
<script lang="ts">
  import { Effect } from "effect"
  
  let count = $state(0)
  let factor = $state(2)
  
  // Derive a computation (not run yet)
  let computation = $derived(
    pipe(
      Effect.sleep("1 second"),
      Effect.map(() => count * factor)
    )
  )
  
  let result = $state<number | null>(null)
  
  // Run computation when it changes
  $effect(() => {
    Effect.runPromise(computation).then(value => {
      result = value
    })
  })
</script>

<p>Result: {result ?? "Computing..."}</p>
```

---

[↑ Back to Top](#table-of-contents)

## Effect with $effect

### Side Effects with Cleanup

```typescript
<script lang="ts">
  import { Effect } from "effect"
  
  let { userId } = $props<{ userId: string }>()
  let userData = $state<User | null>(null)
  
  // Effect runs when userId changes
  $effect(() => {
    let cancelled = false
    
    Effect.runPromise(fetchUser(userId)).then(user => {
      if (!cancelled) {
        userData = user
      }
    })
    
    // Cleanup when effect re-runs or component unmounts
    return () => {
      cancelled = true
    }
  })
</script>
```

### WebSocket with Effect

```typescript
<script lang="ts">
  import { Effect, Stream } from "effect"
  
  let messages = $state<string[]>([])
  
  $effect(() => {
    const program = pipe(
      connectWebSocket(),
      Effect.flatMap((ws) =>
        Stream.runForEach(
          ws.messages,
          (message) => Effect.sync(() => {
            messages = [...messages, message]
          })
        )
      )
    )

    const fiber = Effect.runFork(program)
    
    // Cleanup
    return () => {
      Effect.runFork(
        fiber.pipe(Effect.flatMap(f => f.interrupt))
      )
    }
  })
</script>

<ul>
  {#each messages as message}
    <li>{message}</li>
  {/each}
</ul>
```

### Polling with Effect

```typescript
<script lang="ts">
  import { Effect, Schedule } from "effect"
  
  let stats = $state<Stats | null>(null)
  
  $effect(() => {
    const program = Effect.repeat(
      pipe(
        fetchStats(),
        Effect.map((data) => {
          stats = data
        })
      ),
      Schedule.fixed("5 seconds")
    )

    const fiber = Effect.runFork(program)
    
    return () => {
      Effect.runFork(
        fiber.pipe(Effect.flatMap(f => f.interrupt))
      )
    }
  })
</script>
```

---

[↑ Back to Top](#table-of-contents)

## SvelteKit Integration

### Server Load Functions

```typescript
// src/routes/users/[id]/+page.ts
import { Effect } from "effect"
import { error } from "@sveltejs/kit"
import type { PageLoad } from "./$types"

export const load: PageLoad = async ({ params, fetch }) => {
  const result = await Effect.runPromiseExit(
    pipe(
      fetchUser(params.id),
      Effect.flatMap((user) =>
        pipe(
          fetchUserPosts(params.id),
          Effect.map((posts) => ({ user, posts }))
        )
      )
    )
  )

  if (result._tag === "Success") {
    return result.value
  } else {
    throw error(404, "User not found")
  }
}
```

```svelte
<!-- src/routes/users/[id]/+page.svelte -->
<script lang="ts">
  import type { PageData } from "./$types"
  
  let { data } = $props<{ data: PageData }>()
</script>

<div>
  <h1>{data.user.name}</h1>
  
  <h2>Posts</h2>
  <ul>
    {#each data.posts as post}
      <li>{post.title}</li>
    {/each}
  </ul>
</div>
```

### Server Actions with Effect

```typescript
// src/routes/posts/+page.server.ts
import { Effect } from "effect"
import { fail } from "@sveltejs/kit"
import type { Actions } from "./$types"

export const actions = {
  create: async ({ request }) => {
    const formData = await request.formData()
    const title = formData.get("title") as string
    const content = formData.get("content") as string

    const result = await Effect.runPromiseExit(
      pipe(
        validatePost({ title, content }),
        Effect.flatMap((validated) => createPost(validated))
      )
    )

    if (result._tag === "Success") {
      return { success: true, post: result.value }
    } else {
      return fail(400, {
        error: result.cause._tag,
        message: "Failed to create post"
      })
    }
  }
} satisfies Actions
```

```svelte
<!-- src/routes/posts/+page.svelte -->
<script lang="ts">
  import { enhance } from "$app/forms"
  import type { ActionData } from "./$types"
  
  let { form } = $props<{ form: ActionData }>()
</script>

<form method="POST" action="?/create" use:enhance>
  <input name="title" placeholder="Title" required />
  <textarea name="content" placeholder="Content" required></textarea>
  
  {#if form?.error}
    <p class="error">{form.message}</p>
  {/if}
  
  <button type="submit">Create Post</button>
</form>
```

### API Routes with Effect

```typescript
// src/routes/api/users/+server.ts
import { Effect } from "effect"
import { json, error } from "@sveltejs/kit"
import type { RequestHandler } from "./$types"

export const GET: RequestHandler = async ({ url }) => {
  const limit = parseInt(url.searchParams.get("limit") ?? "10")

  const result = await Effect.runPromiseExit(
    fetchUsers({ limit })
  )

  if (result._tag === "Success") {
    return json(result.value)
  } else {
    throw error(500, "Failed to fetch users")
  }
}

export const POST: RequestHandler = async ({ request }) => {
  const body = await request.json()

  const result = await Effect.runPromiseExit(
    pipe(
      validateUser(body),
      Effect.flatMap((validated) => createUser(validated))
    )
  )

  if (result._tag === "Success") {
    return json(result.value, { status: 201 })
  } else {
    throw error(400, "Invalid user data")
  }
}
```

---

[↑ Back to Top](#table-of-contents)

## Complete Examples

### Example 1: Todo List with Runes

```svelte
<!-- TodoList.svelte -->
<script lang="ts">
  import { Effect, pipe } from "effect"
  
  interface Todo {
    id: string
    text: string
    completed: boolean
  }
  
  type TodoError = 
    | { _tag: "FetchFailed" }
    | { _tag: "SaveFailed" }
  
  let todos = $state<Todo[]>([])
  let loading = $state(false)
  let error = $state<TodoError | null>(null)
  let newTodoText = $state("")
  
  // Fetch todos on mount
  $effect(() => {
    loading = true
    
    Effect.runPromise(
      Effect.tryPromise({
        try: () => fetch("/api/todos").then(r => r.json()),
        catch: () => ({ _tag: "FetchFailed" as const })
      })
    )
      .then(data => {
        todos = data
        loading = false
      })
      .catch(err => {
        error = err
        loading = false
      })
  })
  
  // Add todo
  async function addTodo() {
    if (!newTodoText.trim()) return
    
    loading = true
    
    const result = await Effect.runPromiseExit(
      Effect.tryPromise({
        try: () =>
          fetch("/api/todos", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ text: newTodoText })
          }).then(r => r.json()),
        catch: () => ({ _tag: "SaveFailed" as const })
      })
    )
    
    if (result._tag === "Success") {
      todos = [...todos, result.value]
      newTodoText = ""
    } else {
      error = result.cause
    }
    
    loading = false
  }
  
  // Toggle todo
  async function toggleTodo(id: string) {
    const todo = todos.find(t => t.id === id)
    if (!todo) return
    
    const result = await Effect.runPromiseExit(
      Effect.tryPromise({
        try: () =>
          fetch(`/api/todos/${id}`, {
            method: "PATCH",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ completed: !todo.completed })
          }).then(r => r.json()),
        catch: () => ({ _tag: "SaveFailed" as const })
      })
    )
    
    if (result._tag === "Success") {
      todos = todos.map(t => (t.id === id ? result.value : t))
    } else {
      error = result.cause
    }
  }
  
  // Derived computed values
  let activeTodos = $derived(todos.filter(t => !t.completed))
  let completedTodos = $derived(todos.filter(t => t.completed))
  let allCompleted = $derived(todos.length > 0 && activeTodos.length === 0)
</script>

<div class="todo-app">
  <h1>Todos</h1>
  
  {#if error}
    <div class="error">{error._tag}</div>
  {/if}
  
  <form onsubmit={(e) => { e.preventDefault(); addTodo() }}>
    <input
      bind:value={newTodoText}
      placeholder="What needs to be done?"
      disabled={loading}
    />
    <button type="submit" disabled={loading || !newTodoText.trim()}>
      Add
    </button>
  </form>
  
  {#if loading && todos.length === 0}
    <p>Loading...</p>
  {:else}
    <div class="todos">
      <h2>Active ({activeTodos.length})</h2>
      <ul>
        {#each activeTodos as todo (todo.id)}
          <li>
            <input
              type="checkbox"
              checked={todo.completed}
              onchange={() => toggleTodo(todo.id)}
            />
            <span>{todo.text}</span>
          </li>
        {/each}
      </ul>
      
      {#if completedTodos.length > 0}
        <h2>Completed ({completedTodos.length})</h2>
        <ul>
          {#each completedTodos as todo (todo.id)}
            <li class="completed">
              <input
                type="checkbox"
                checked={todo.completed}
                onchange={() => toggleTodo(todo.id)}
              />
              <span>{todo.text}</span>
            </li>
          {/each}
        </ul>
      {/if}
    </div>
  {/if}
  
  {#if allCompleted}
    <p class="success">All done! 🎉</p>
  {/if}
</div>

<style>
  .todo-app {
    max-width: 600px;
    margin: 0 auto;
    padding: 2rem;
  }
  
  .error {
    color: red;
    padding: 1rem;
    background: #fee;
    border-radius: 4px;
  }
  
  form {
    display: flex;
    gap: 0.5rem;
    margin: 1rem 0;
  }
  
  input[type="text"] {
    flex: 1;
    padding: 0.5rem;
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
  
  li.completed span {
    text-decoration: line-through;
    color: #999;
  }
  
  .success {
    color: green;
    font-weight: bold;
  }
</style>
```

### Example 2: Real-Time Dashboard

```svelte
<!-- Dashboard.svelte -->
<script lang="ts">
  import { Effect, Stream, Schedule } from "effect"
  
  interface Stats {
    users: number
    posts: number
    activeNow: number
  }
  
  let stats = $state<Stats | null>(null)
  let lastUpdated = $state<Date | null>(null)
  let connected = $state(false)
  
  // Poll stats every 5 seconds
  $effect(() => {
    connected = true

    const program = Effect.repeat(
      pipe(
        Effect.tryPromise({
          try: () => fetch("/api/stats").then(r => r.json()),
          catch: (error) => ({ _tag: "FetchFailed" as const, error })
        }),
        Effect.map((data) => {
          stats = data
          lastUpdated = new Date()
        })
      ),
      Schedule.fixed("5 seconds")
    )

    const fiber = Effect.runFork(program)
    
    // Cleanup
    return () => {
      connected = false
      Effect.runFork(
        fiber.pipe(Effect.flatMap(f => f.interrupt))
      )
    }
  })
  
  // Derived values
  let statusColor = $derived(connected ? "green" : "red")
  let formattedTime = $derived(
    lastUpdated?.toLocaleTimeString() ?? "Never"
  )
</script>

<div class="dashboard">
  <div class="header">
    <h1>Dashboard</h1>
    <div class="status" style="color: {statusColor}">
      {connected ? "●" : "○"} {connected ? "Connected" : "Disconnected"}
    </div>
  </div>
  
  {#if stats}
    <div class="stats">
      <div class="stat">
        <h2>{stats.users}</h2>
        <p>Total Users</p>
      </div>
      
      <div class="stat">
        <h2>{stats.posts}</h2>
        <p>Total Posts</p>
      </div>
      
      <div class="stat">
        <h2>{stats.activeNow}</h2>
        <p>Active Now</p>
      </div>
    </div>
    
    <p class="updated">Last updated: {formattedTime}</p>
  {:else}
    <p>Loading stats...</p>
  {/if}
</div>

<style>
  .dashboard {
    padding: 2rem;
  }
  
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
  }
  
  .stats {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 1rem;
  }
  
  .stat {
    padding: 1.5rem;
    background: #f5f5f5;
    border-radius: 8px;
    text-align: center;
  }
  
  .stat h2 {
    font-size: 2.5rem;
    margin: 0;
  }
  
  .updated {
    margin-top: 1rem;
    color: #666;
    font-size: 0.9rem;
  }
</style>
```

### Example 3: Form with Validation

```svelte
<!-- SignupForm.svelte -->
<script lang="ts">
  import { Effect, pipe } from "effect"
  
  type ValidationError =
    | { _tag: "Required"; field: string }
    | { _tag: "InvalidEmail"; field: string }
    | { _tag: "TooShort"; field: string; min: number }
    | { _tag: "PasswordMismatch" }
  
  let email = $state("")
  let password = $state("")
  let confirmPassword = $state("")
  let touched = $state({ email: false, password: false, confirmPassword: false })
  
  // Validation effects
  const validateEmail = (value: string): Effect.Effect<string, ValidationError> =>
    value.trim().length === 0
      ? Effect.fail({ _tag: "Required", field: "email" })
      : /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)
      ? Effect.succeed(value)
      : Effect.fail({ _tag: "InvalidEmail", field: "email" })
  
  const validatePassword = (value: string): Effect.Effect<string, ValidationError> =>
    value.length === 0
      ? Effect.fail({ _tag: "Required", field: "password" })
      : value.length < 8
      ? Effect.fail({ _tag: "TooShort", field: "password", min: 8 })
      : Effect.succeed(value)
  
  // Derived validation state
  let emailErrors = $derived.by(() => {
    if (!touched.email || !email) return []
    
    const result = Effect.runSyncExit(validateEmail(email))
    return result._tag === "Failure" ? [result.cause] : []
  })
  
  let passwordErrors = $derived.by(() => {
    if (!touched.password || !password) return []
    
    const result = Effect.runSyncExit(validatePassword(password))
    return result._tag === "Failure" ? [result.cause] : []
  })
  
  let confirmErrors = $derived.by(() => {
    if (!touched.confirmPassword) return []
    
    return password !== confirmPassword
      ? [{ _tag: "PasswordMismatch" as const }]
      : []
  })
  
  let isValid = $derived(
    emailErrors.length === 0 &&
    passwordErrors.length === 0 &&
    confirmErrors.length === 0 &&
    email.length > 0 &&
    password.length > 0 &&
    confirmPassword.length > 0
  )
  
  let submitting = $state(false)
  let submitError = $state<string | null>(null)
  
  async function handleSubmit() {
    // Mark all as touched
    touched = { email: true, password: true, confirmPassword: true }
    
    if (!isValid) return
    
    submitting = true
    submitError = null
    
    const result = await Effect.runPromiseExit(
      Effect.tryPromise({
        try: () =>
          fetch("/api/signup", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ email, password })
          }),
        catch: () => ({ _tag: "SignupFailed" as const })
      })
    )
    
    if (result._tag === "Success") {
      // Success - redirect or show message
      console.log("Signup successful!")
    } else {
      submitError = "Signup failed. Please try again."
    }
    
    submitting = false
  }
</script>

<form onsubmit={(e) => { e.preventDefault(); handleSubmit() }}>
  <h1>Sign Up</h1>
  
  {#if submitError}
    <div class="error">{submitError}</div>
  {/if}
  
  <div class="field">
    <label for="email">Email</label>
    <input
      id="email"
      type="email"
      bind:value={email}
      onblur={() => touched.email = true}
      aria-invalid={emailErrors.length > 0}
    />
    {#if touched.email && emailErrors.length > 0}
      <span class="error">
        {#each emailErrors as error}
          {#if error._tag === "Required"}Required field{/if}
          {#if error._tag === "InvalidEmail"}Invalid email address{/if}
        {/each}
      </span>
    {/if}
  </div>
  
  <div class="field">
    <label for="password">Password</label>
    <input
      id="password"
      type="password"
      bind:value={password}
      onblur={() => touched.password = true}
      aria-invalid={passwordErrors.length > 0}
    />
    {#if touched.password && passwordErrors.length > 0}
      <span class="error">
        {#each passwordErrors as error}
          {#if error._tag === "Required"}Required field{/if}
          {#if error._tag === "TooShort"}Minimum {error.min} characters{/if}
        {/each}
      </span>
    {/if}
  </div>
  
  <div class="field">
    <label for="confirmPassword">Confirm Password</label>
    <input
      id="confirmPassword"
      type="password"
      bind:value={confirmPassword}
      onblur={() => touched.confirmPassword = true}
      aria-invalid={confirmErrors.length > 0}
    />
    {#if touched.confirmPassword && confirmErrors.length > 0}
      <span class="error">Passwords do not match</span>
    {/if}
  </div>
  
  <button type="submit" disabled={!isValid || submitting}>
    {submitting ? "Signing up..." : "Sign Up"}
  </button>
</form>

<style>
  form {
    max-width: 400px;
    margin: 2rem auto;
    padding: 2rem;
    border: 1px solid #ddd;
    border-radius: 8px;
  }
  
  .field {
    margin-bottom: 1rem;
  }
  
  label {
    display: block;
    margin-bottom: 0.25rem;
    font-weight: 500;
  }
  
  input {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
  
  input[aria-invalid="true"] {
    border-color: red;
  }
  
  .error {
    color: red;
    font-size: 0.875rem;
    margin-top: 0.25rem;
  }
  
  button {
    width: 100%;
    padding: 0.75rem;
    background: #0066cc;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 1rem;
    cursor: pointer;
  }
  
  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
```

---

[↑ Back to Top](#table-of-contents)

## Best Practices

### 1. Use $effect for Effect Execution

```typescript
// ✅ Good - Effect runs when dependencies change
$effect(() => {
  Effect.runPromise(fetchUser(userId)).then(setUser)
})

// ❌ Bad - Effect doesn't track userId changes
onMount(() => {
  Effect.runPromise(fetchUser(userId)).then(setUser)
})
```

### 2. Cleanup in $effect

```typescript
// ✅ Good - Proper cleanup
$effect(() => {
  const program = startPolling()
  const fiber = Effect.runFork(program)
  
  return () => {
    Effect.runFork(fiber.pipe(Effect.flatMap(f => f.interrupt)))
  }
})
```

### 3. Use $derived for Pure Computations

```typescript
// ✅ Good - Pure derived value
let doubled = $derived(count * 2)

// ❌ Bad - Side effect in derived
let doubled = $derived.by(() => {
  Effect.runPromise(logValue(count)) // Side effect!
  return count * 2
})
```

### 4. Separate State and Effects

```typescript
// ✅ Good - Clear separation
let data = $state<User | null>(null)

$effect(() => {
  Effect.runPromise(fetchUser(id)).then(user => {
    data = user
  })
})

// ❌ Bad - Mixing concerns
let data = $derived.by(() => {
  Effect.runPromise(fetchUser(id)) // Async in derived!
})
```

### 5. Type Your State Properly

```typescript
// ✅ Good - Explicit types
let user = $state<User | null>(null)
let state = $state<AsyncState<User, UserError>>({ status: "idle" })

// ❌ Bad - Implicit any
let user = $state(null)
```

---

[↑ Back to Top](#table-of-contents)

## Migration from Stores

If you're migrating from the old stores API:

```typescript
// Old (Stores)
const count = writable(0)
const doubled = derived(count, $count => $count * 2)

$: console.log($count)

// New (Runes)
let count = $state(0)
let doubled = $derived(count * 2)

$effect(() => {
  console.log(count)
})
```

---

This guide shows the modern way to integrate Effect-TS with Svelte 5's runes system, providing reactive, type-safe state management for your frontend applications!
