# End-to-End Project: Task Management App with Svelte 5 + Effect-TS

**Complete Production-Ready Application**

This is a complete, production-ready task management application built with SvelteKit, Svelte 5 runes, and Effect-TS. It demonstrates all the patterns from the course in a real-world context.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Project Setup](#project-setup)
3. [Project Structure](#project-structure)
4. [Domain Layer](#domain-layer)
5. [Service Layer](#service-layer)
6. [Effect Layers](#effect-layers)
7. [Components](#components)
8. [Routes](#routes)
9. [Utilities](#utilities)
10. [Running the Project](#running-the-project)

---

## Project Overview

### Features

- ✅ Task CRUD operations with validation
- ✅ Project organization
- ✅ Real-time collaboration (WebSocket)
- ✅ Optimistic updates
- ✅ Offline support with queue
- ✅ Search and filtering
- ✅ User authentication
- ✅ Analytics dashboard
- ✅ Dark mode
- ✅ Full TypeScript type safety

### Tech Stack

- **Frontend**: SvelteKit + Svelte 5 (runes)
- **State Management**: Effect-TS
- **Styling**: Tailwind CSS
- **Validation**: Effect Schema
- **Testing**: Vitest + Testing Library

---

[↑ Back to Top](#table-of-contents)

## Project Setup

### Installation

```bash
# Create new SvelteKit project
npm create svelte@latest taskmaster
cd taskmaster

# Install dependencies
npm install

# Install Effect-TS
npm install effect @effect/platform @effect/schema

# Install Tailwind CSS
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# Install dev dependencies
npm install -D vitest @testing-library/svelte @testing-library/jest-dom
```

### Configuration

**svelte.config.js**
```javascript
import adapter from '@sveltejs/adapter-auto'
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte'

/** @type {import('@sveltejs/kit').Config} */
const config = {
  preprocess: vitePreprocess(),
  kit: {
    adapter: adapter(),
    alias: {
      '$lib': './src/lib',
      '$domain': './src/lib/domain',
      '$services': './src/lib/services',
      '$effects': './src/lib/effects'
    }
  }
}

export default config
```

**tailwind.config.js**
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./src/**/*.{html,js,svelte,ts}'],
  theme: {
    extend: {}
  },
  plugins: []
}
```

---

[↑ Back to Top](#table-of-contents)

## Project Structure

```
taskmaster/
├── src/
│   ├── lib/
│   │   ├── domain/
│   │   │   ├── Task.ts              # Task domain model
│   │   │   ├── Project.ts           # Project domain model
│   │   │   ├── User.ts              # User domain model
│   │   │   └── errors.ts            # Domain error types
│   │   ├── services/
│   │   │   ├── ApiClient.ts         # HTTP client
│   │   │   ├── TaskService.ts       # Task operations
│   │   │   ├── ProjectService.ts    # Project operations
│   │   │   ├── AuthService.ts       # Authentication
│   │   │   ├── WebSocketService.ts  # Real-time updates
│   │   │   └── OfflineQueue.ts      # Offline support
│   │   ├── effects/
│   │   │   ├── runtime.ts           # Effect runtime
│   │   │   └── layers.ts            # Service composition
│   │   ├── stores/
│   │   │   └── effect-runes.svelte.ts  # Reusable runes
│   │   └── components/
│   │       ├── TaskList.svelte
│   │       ├── TaskForm.svelte
│   │       ├── ProjectSelector.svelte
│   │       └── Dashboard.svelte
│   ├── routes/
│   │   ├── +layout.svelte
│   │   ├── +page.svelte             # Dashboard
│   │   ├── tasks/
│   │   │   ├── +page.svelte         # Task list
│   │   │   ├── [id]/+page.svelte    # Task detail
│   │   │   └── [id]/+page.ts        # Task loader
│   │   ├── projects/
│   │   │   ├── +page.svelte
│   │   │   └── [id]/+page.svelte
│   │   └── api/
│   │       ├── tasks/
│   │       │   ├── +server.ts
│   │       │   └── [id]/+server.ts
│   │       └── projects/
│   │           └── +server.ts
│   └── app.css                      # Tailwind imports
├── package.json
├── svelte.config.js
├── tailwind.config.js
├── tsconfig.json
└── vite.config.ts
```

---

[↑ Back to Top](#table-of-contents)

## Domain Layer

### Task Model

**src/lib/domain/Task.ts**
```typescript
import { Schema } from "@effect/schema"
import { Effect } from "effect"

// Task status enum
export const TaskStatus = Schema.Literal("todo", "in_progress", "done")
export type TaskStatus = Schema.Schema.Type<typeof TaskStatus>
// export type TaskStatus = typeof TaskStatus.Type  // Shorter with less noise!

// Task priority enum
export const TaskPriority = Schema.Literal("low", "medium", "high")
export type TaskPriority = Schema.Schema.Type<typeof TaskPriority>

// Task schema
export const Task = Schema.Struct({
  id: Schema.String,
  title: Schema.String,
  description: Schema.optional(Schema.String),
  status: TaskStatus,
  priority: TaskPriority,
  projectId: Schema.String,
  assigneeId: Schema.optional(Schema.String),
  dueDate: Schema.optional(Schema.Date),
  createdAt: Schema.Date,
  updatedAt: Schema.Date,
  _optimistic: Schema.optional(Schema.Boolean)
})

export type Task = Schema.Schema.Type<typeof Task>

// Create task input
export const CreateTaskInput = Schema.Struct({
  title: Schema.String.pipe(
    Schema.minLength(1, { message: () => "Title is required" }),
    Schema.maxLength(200, { message: () => "Title too long" })
  ),
  description: Schema.optional(Schema.String),
  status: Schema.optional(TaskStatus),
  priority: TaskPriority,
  projectId: Schema.String,
  assigneeId: Schema.optional(Schema.String),
  dueDate: Schema.optional(Schema.Date)
})

export type CreateTaskInput = Schema.Schema.Type<typeof CreateTaskInput>

// Update task input
export const UpdateTaskInput = Schema.Struct({
  title: Schema.optional(Schema.String.pipe(
    Schema.minLength(1),
    Schema.maxLength(200)
  )),
  description: Schema.optional(Schema.String),
  status: Schema.optional(TaskStatus),
  priority: Schema.optional(TaskPriority),
  assigneeId: Schema.optional(Schema.String),
  dueDate: Schema.optional(Schema.NullOr(Schema.Date))
})

export type UpdateTaskInput = Schema.Schema.Type<typeof UpdateTaskInput>

// Validation
export const validateTask = Schema.decode(Task)
export const validateCreateTaskInput = Schema.decode(CreateTaskInput)
export const validateUpdateTaskInput = Schema.decode(UpdateTaskInput)
```

### Project Model

**src/lib/domain/Project.ts**
```typescript
import { Schema } from "@effect/schema"

export const Project = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  description: Schema.optional(Schema.String),
  color: Schema.String,
  ownerId: Schema.String,
  createdAt: Schema.Date,
  updatedAt: Schema.Date
})

export type Project = Schema.Schema.Type<typeof Project>

export const CreateProjectInput = Schema.Struct({
  name: Schema.String.pipe(
    Schema.minLength(1, { message: () => "Name is required" }),
    Schema.maxLength(100, { message: () => "Name too long" })
  ),
  description: Schema.optional(Schema.String),
  color: Schema.String
})

export type CreateProjectInput = Schema.Schema.Type<typeof CreateProjectInput>

export const validateProject = Schema.decode(Project)
export const validateCreateProjectInput = Schema.decode(CreateProjectInput)
```

### User Model

**src/lib/domain/User.ts**
```typescript
import { Schema } from "@effect/schema"

export const User = Schema.Struct({
  id: Schema.String,
  email: Schema.String,
  name: Schema.String,
  avatar: Schema.optional(Schema.String),
  createdAt: Schema.Date
})

export type User = Schema.Schema.Type<typeof User>

export const validateUser = Schema.decode(User)
```

### Error Types

**src/lib/domain/errors.ts**
```typescript
export type TaskError =
  | { _tag: "TaskNotFound"; id: string }
  | { _tag: "InvalidTaskData"; errors: unknown }
  | { _tag: "Unauthorized" }

export type ProjectError =
  | { _tag: "ProjectNotFound"; id: string }
  | { _tag: "InvalidProjectData"; errors: unknown }
  | { _tag: "Unauthorized" }

export type ApiError =
  | { _tag: "NetworkError"; cause: unknown }
  | { _tag: "InvalidResponse"; status: number }
  | { _tag: "Timeout" }

export type AuthError =
  | { _tag: "InvalidCredentials" }
  | { _tag: "SessionExpired" }
  | { _tag: "Unauthorized" }
```

---

[↑ Back to Top](#table-of-contents)

## Service Layer

### API Client

**src/lib/services/ApiClient.ts**
```typescript
import { Effect, Context, Layer } from "effect"
import type { ApiError } from "$domain/errors"

export interface ApiClient {
  readonly get: <T>(url: string) => Effect.Effect<T, ApiError>
  readonly post: <T, B>(url: string, body: B) => Effect.Effect<T, ApiError>
  readonly put: <T, B>(url: string, body: B) => Effect.Effect<T, ApiError>
  readonly delete: (url: string) => Effect.Effect<void, ApiError>
}

export class ApiClient extends Context.Tag("ApiClient")<
  ApiClient,
  ApiClient
>() {}

interface Config {
  readonly baseUrl: string
  readonly timeout: number
}

const makeApiClient = (config: Config): ApiClient => ({
  get: <T>(url: string) =>
    Effect.tryPromise({
      try: async () => {
        const controller = new AbortController()
        const timeout = setTimeout(() => controller.abort(), config.timeout)
        
        const response = await fetch(`${config.baseUrl}${url}`, {
          signal: controller.signal,
          credentials: "include"
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
        const response = await fetch(`${config.baseUrl}${url}`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          credentials: "include",
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
    }),
  
  put: <T, B>(url: string, body: B) =>
    Effect.tryPromise({
      try: async () => {
        const response = await fetch(`${config.baseUrl}${url}`, {
          method: "PUT",
          headers: { "Content-Type": "application/json" },
          credentials: "include",
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
    }),
  
  delete: (url: string) =>
    Effect.tryPromise({
      try: async () => {
        const response = await fetch(`${config.baseUrl}${url}`, {
          method: "DELETE",
          credentials: "include"
        })
        
        if (!response.ok) {
          throw { status: response.status }
        }
      },
      catch: (error) =>
        typeof error === "object" && error !== null && "status" in error
          ? { _tag: "InvalidResponse" as const, status: error.status as number }
          : { _tag: "NetworkError" as const, cause: error }
    })
})

export const ApiClientLive = Layer.succeed(
  ApiClient,
  makeApiClient({
    baseUrl: import.meta.env.VITE_API_URL || "/api",
    timeout: 30000
  })
)
```

### Task Service

**src/lib/services/TaskService.ts**
```typescript
import { Effect, Context, Layer, pipe, Ref } from "effect"
import { ApiClient } from "./ApiClient"
import { validateTask, validateCreateTaskInput } from "$domain/Task"
import type { Task, CreateTaskInput, UpdateTaskInput } from "$domain/Task"
import type { TaskError } from "$domain/errors"

export interface TaskService {
  readonly tasks: Ref.Ref<Task[]>
  readonly listTasks: (projectId?: string) => Effect.Effect<Task[], TaskError>
  readonly getTask: (id: string) => Effect.Effect<Task, TaskError>
  readonly createTask: (input: CreateTaskInput) => Effect.Effect<Task, TaskError>
  readonly updateTask: (id: string, input: UpdateTaskInput) => Effect.Effect<Task, TaskError>
  readonly deleteTask: (id: string) => Effect.Effect<void, TaskError>
}

export class TaskService extends Context.Tag("TaskService")<
  TaskService,
  TaskService
>() {}

export const TaskServiceLive = Layer.effect(
  TaskService,
  Effect.gen(function* () {
    const api = yield* ApiClient
    const tasks = yield* Ref.make<Task[]>([])

    return {
    tasks,
    
    listTasks: (projectId?: string) =>
      pipe(
        api.get<unknown[]>(
          projectId ? `/tasks?projectId=${projectId}` : "/tasks"
        ),
        Effect.flatMap((data) =>
          Effect.all(data.map(validateTask), { concurrency: "unbounded" })
        ),
        Effect.tap((loadedTasks) => Ref.set(tasks, loadedTasks)),
        Effect.mapError(() => ({ _tag: "TaskNotFound" as const, id: "all" }))
      ),
    
    getTask: (id) =>
      pipe(
        api.get<unknown>(`/tasks/${id}`),
        Effect.flatMap(validateTask),
        Effect.mapError(() => ({ _tag: "TaskNotFound" as const, id }))
      ),
    
    createTask: (input) => {
      // Optimistic task
      const optimisticTask: Task = {
        id: `temp-${Date.now()}`,
        title: input.title,
        description: input.description,
        status: input.status || "todo",
        priority: input.priority,
        projectId: input.projectId,
        assigneeId: input.assigneeId,
        dueDate: input.dueDate,
        createdAt: new Date(),
        updatedAt: new Date(),
        _optimistic: true
      }
      
      return pipe(
        // Add optimistically
        Ref.update(tasks, (ts) => [...ts, optimisticTask]),
        Effect.flatMap(() =>
          pipe(
            validateCreateTaskInput(input),
            Effect.flatMap((validated) => api.post<unknown>("/tasks", validated)),
            Effect.flatMap(validateTask)
          )
        ),
        Effect.tap((savedTask) =>
          // Replace optimistic with real
          Ref.update(tasks, (ts) =>
            ts.map((t) => (t.id === optimisticTask.id ? savedTask : t))
          )
        ),
        Effect.tapError(() =>
          // Rollback on error
          Ref.update(tasks, (ts) =>
            ts.filter((t) => t.id !== optimisticTask.id)
          )
        ),
        Effect.mapError(() => ({
          _tag: "InvalidTaskData" as const,
          errors: "Failed to create task"
        }))
      )
    },
    
    updateTask: (id, input) =>
      pipe(
        api.put<unknown>(`/tasks/${id}`, input),
        Effect.flatMap(validateTask),
        Effect.tap((updatedTask) =>
          Ref.update(tasks, (ts) =>
            ts.map((t) => (t.id === id ? updatedTask : t))
          )
        ),
        Effect.mapError(() => ({ _tag: "TaskNotFound" as const, id }))
      ),
    
    deleteTask: (id) =>
      pipe(
        Ref.get(tasks),
        Effect.flatMap((ts) => {
          const task = ts.find((t) => t.id === id)
          if (!task) {
            return Effect.fail({ _tag: "TaskNotFound" as const, id })
          }
          
          return pipe(
            // Remove optimistically
            Ref.update(tasks, (ts) => ts.filter((t) => t.id !== id)),
            Effect.flatMap(() => api.delete(`/tasks/${id}`)),
            Effect.tapError(() =>
              // Rollback on error
              Ref.update(tasks, (ts) => [...ts, task])
            )
          )
        })
      )
    };
  })
)
```

### Project Service

**src/lib/services/ProjectService.ts**
```typescript
import { Effect, Context, Layer, pipe, Ref } from "effect"
import { ApiClient } from "./ApiClient"
import { validateProject, validateCreateProjectInput } from "$domain/Project"
import type { Project, CreateProjectInput } from "$domain/Project"
import type { ProjectError } from "$domain/errors"

export interface ProjectService {
  readonly projects: Ref.Ref<Project[]>
  readonly listProjects: () => Effect.Effect<Project[], ProjectError>
  readonly getProject: (id: string) => Effect.Effect<Project, ProjectError>
  readonly createProject: (input: CreateProjectInput) => Effect.Effect<Project, ProjectError>
  readonly deleteProject: (id: string) => Effect.Effect<void, ProjectError>
}

export class ProjectService extends Context.Tag("ProjectService")<
  ProjectService,
  ProjectService
>() {}

export const ProjectServiceLive = Layer.effect(
  ProjectService,
  Effect.gen(function* () {
    const api = yield* ApiClient
    const projects = yield* Ref.make<Project[]>([])

    return {
    projects,
    
    listProjects: () =>
      pipe(
        api.get<unknown[]>("/projects"),
        Effect.flatMap((data) =>
          Effect.all(data.map(validateProject), { concurrency: "unbounded" })
        ),
        Effect.tap((loadedProjects) => Ref.set(projects, loadedProjects)),
        Effect.mapError(() => ({
          _tag: "ProjectNotFound" as const,
          id: "all"
        }))
      ),
    
    getProject: (id) =>
      pipe(
        api.get<unknown>(`/projects/${id}`),
        Effect.flatMap(validateProject),
        Effect.mapError(() => ({ _tag: "ProjectNotFound" as const, id }))
      ),
    
    createProject: (input) =>
      pipe(
        validateCreateProjectInput(input),
        Effect.flatMap((validated) =>
          api.post<unknown>("/projects", validated)
        ),
        Effect.flatMap(validateProject),
        Effect.tap((newProject) =>
          Ref.update(projects, (ps) => [...ps, newProject])
        ),
        Effect.mapError(() => ({
          _tag: "InvalidProjectData" as const,
          errors: "Failed to create project"
        }))
      ),
    
    deleteProject: (id) =>
      pipe(
        api.delete(`/projects/${id}`),
        Effect.tap(() =>
          Ref.update(projects, (ps) => ps.filter((p) => p.id !== id))
        ),
        Effect.mapError(() => ({ _tag: "ProjectNotFound" as const, id }))
      )
    };
  })
)
```

### WebSocket Service

**src/lib/services/WebSocketService.ts**
```typescript
import { Effect, Context, Layer, Stream, Queue } from "effect"

interface WebSocketMessage {
  type: "task_updated" | "task_created" | "task_deleted"
  data: unknown
}

export interface WebSocketService {
  readonly messages: Stream.Stream<WebSocketMessage>
  readonly send: (message: WebSocketMessage) => Effect.Effect<void>
}

export class WebSocketService extends Context.Tag("WebSocketService")<
  WebSocketService,
  WebSocketService
>() {}

const makeWebSocketService = Effect.gen(function* () {
  const queue = yield* Queue.unbounded<WebSocketMessage>()
  
  // Connect WebSocket
  const ws = yield* Effect.sync(
    () => new WebSocket(import.meta.env.VITE_WS_URL || "ws://localhost:3000/ws")
  )
  
  // Wait for connection
  yield* Effect.async<void>((resume) => {
    ws.onopen = () => resume(Effect.void)
    ws.onerror = () => resume(Effect.void)
  })
  
  // Handle incoming messages
  yield* Effect.sync(() => {
    ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data) as WebSocketMessage
        Effect.runFork(Queue.offer(queue, message))
      } catch (error) {
        console.error("Failed to parse WebSocket message:", error)
      }
    }
  })
  
  // Register cleanup
  yield* Effect.addFinalizer(() =>
    Effect.sync(() => {
      ws.close()
    })
  )

  return {
    messages: Stream.fromQueue(queue),

    send: (message) =>
      Effect.sync(() => {
        ws.send(JSON.stringify(message))
      })
  }
})

export const WebSocketServiceLive = Layer.scoped(
  WebSocketService,
  makeWebSocketService
)
```

---

[↑ Back to Top](#table-of-contents)

## Effect Layers

**src/lib/effects/layers.ts**
```typescript
import { Layer } from "effect"
import { ApiClientLive } from "$services/ApiClient"
import { TaskServiceLive } from "$services/TaskService"
import { ProjectServiceLive } from "$services/ProjectService"
import { WebSocketServiceLive } from "$services/WebSocketService"

// Infrastructure layer
const InfrastructureLayer = Layer.merge(ApiClientLive, WebSocketServiceLive)

// Service layer
const ServiceLayer = Layer.mergeAll(TaskServiceLive, ProjectServiceLive).pipe(
  Layer.provide(InfrastructureLayer)
)

// Complete application layer
export const AppLayer = ServiceLayer
```

**src/lib/effects/runtime.ts**
```typescript
import { Effect, Layer, Runtime } from "effect"
import { AppLayer } from "./layers"

// Create runtime with all services
export const AppRuntime = Layer.toRuntime(AppLayer).pipe(Effect.runSync)
```

---

[↑ Back to Top](#table-of-contents)

## Utilities

### Effect Runes

**src/lib/stores/effect-runes.svelte.ts**
```typescript
import { Effect, Exit } from "effect"
import { AppRuntime } from "$effects/runtime"

export type AsyncState<T, E> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: E }

export function createEffectState<A, E>(
  effect: Effect.Effect<A, E, never>
) {
  let state = $state<AsyncState<A, E>>({ status: "loading" })
  
  Effect.runPromiseExit(effect, { runtime: AppRuntime }).then((exit) => {
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

export function createRefreshableEffect<A, E>(
  effectFn: () => Effect.Effect<A, E, never>
) {
  let state = $state<AsyncState<A, E>>({ status: "loading" })
  
  const load = () => {
    state = { status: "loading" }
    Effect.runPromiseExit(effectFn(), { runtime: AppRuntime }).then((exit) => {
      if (Exit.isSuccess(exit)) {
        state = { status: "success", data: exit.value }
      } else {
        state = { status: "error", error: Exit.unannotate(exit.cause) }
      }
    })
  }
  
  load() // Initial load
  
  return {
    get current() {
      return state
    },
    refresh: load
  }
}
```

---

[↑ Back to Top](#table-of-contents)

## Components

### Task List

**src/lib/components/TaskList.svelte**
```svelte
<script lang="ts">
  import { Effect, Ref } from "effect"
  import { TaskService } from "$services/TaskService"
  import { AppRuntime } from "$effects/runtime"
  import type { Task } from "$domain/Task"
  
  let { projectId } = $props<{ projectId?: string }>()
  
  let tasks = $state<Task[]>([])
  let loading = $state(true)
  let error = $state<string | null>(null)
  
  // Load tasks
  $effect(() => {
    loading = true
    error = null
    
    Effect.runPromise(
      Effect.gen(function* () {
        const service = yield* TaskService
        const loadedTasks = yield* service.listTasks(projectId)
        return loadedTasks
      }),
      { runtime: AppRuntime }
    )
      .then((loadedTasks) => {
        tasks = loadedTasks
        loading = false
      })
      .catch((err) => {
        error = String(err)
        loading = false
      })
  })
  
  // Delete task
  async function handleDelete(id: string) {
    const result = await Effect.runPromiseExit(
      Effect.gen(function* () {
        const service = yield* TaskService
        yield* service.deleteTask(id)
      }),
      { runtime: AppRuntime }
    )
    
    if (result._tag === "Success") {
      tasks = tasks.filter((t) => t.id !== id)
    }
  }
  
  // Toggle task status
  async function handleToggle(task: Task) {
    const newStatus = task.status === "done" ? "todo" : "done"
    
    const result = await Effect.runPromiseExit(
      Effect.gen(function* () {
        const service = yield* TaskService
        return yield* service.updateTask(task.id, { status: newStatus })
      }),
      { runtime: AppRuntime }
    )
    
    if (result._tag === "Success") {
      tasks = tasks.map((t) => (t.id === task.id ? result.value : t))
    }
  }
  
  // Group by status
  let todoTasks = $derived(tasks.filter((t) => t.status === "todo"))
  let inProgressTasks = $derived(tasks.filter((t) => t.status === "in_progress"))
  let doneTasks = $derived(tasks.filter((t) => t.status === "done"))
</script>

<div class="task-list">
  {#if loading}
    <div class="loading">
      <div class="spinner"></div>
      <p>Loading tasks...</p>
    </div>
  {:else if error}
    <div class="error">
      <p>Error: {error}</p>
    </div>
  {:else}
    <div class="task-columns">
      <div class="task-column">
        <h2>To Do ({todoTasks.length})</h2>
        {#each todoTasks as task (task.id)}
          <div class="task-card" class:optimistic={task._optimistic}>
            <div class="task-header">
              <input
                type="checkbox"
                checked={task.status === "done"}
                onchange={() => handleToggle(task)}
              />
              <h3>{task.title}</h3>
            </div>
            {#if task.description}
              <p class="task-description">{task.description}</p>
            {/if}
            <div class="task-footer">
              <span class="priority priority-{task.priority}">
                {task.priority}
              </span>
              <button onclick={() => handleDelete(task.id)}>Delete</button>
            </div>
          </div>
        {/each}
      </div>
      
      <div class="task-column">
        <h2>In Progress ({inProgressTasks.length})</h2>
        {#each inProgressTasks as task (task.id)}
          <div class="task-card">
            <div class="task-header">
              <input
                type="checkbox"
                checked={task.status === "done"}
                onchange={() => handleToggle(task)}
              />
              <h3>{task.title}</h3>
            </div>
            {#if task.description}
              <p class="task-description">{task.description}</p>
            {/if}
            <div class="task-footer">
              <span class="priority priority-{task.priority}">
                {task.priority}
              </span>
              <button onclick={() => handleDelete(task.id)}>Delete</button>
            </div>
          </div>
        {/each}
      </div>
      
      <div class="task-column">
        <h2>Done ({doneTasks.length})</h2>
        {#each doneTasks as task (task.id)}
          <div class="task-card done">
            <div class="task-header">
              <input
                type="checkbox"
                checked={task.status === "done"}
                onchange={() => handleToggle(task)}
              />
              <h3>{task.title}</h3>
            </div>
            {#if task.description}
              <p class="task-description">{task.description}</p>
            {/if}
            <div class="task-footer">
              <span class="priority priority-{task.priority}">
                {task.priority}
              </span>
              <button onclick={() => handleDelete(task.id)}>Delete</button>
            </div>
          </div>
        {/each}
      </div>
    </div>
  {/if}
</div>

<style>
  .task-list {
    padding: 1rem;
  }
  
  .loading {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 3rem;
  }
  
  .spinner {
    width: 40px;
    height: 40px;
    border: 4px solid #f3f3f3;
    border-top: 4px solid #3b82f6;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  .error {
    padding: 1rem;
    background: #fee;
    color: #c00;
    border-radius: 8px;
  }
  
  .task-columns {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 1.5rem;
  }
  
  .task-column {
    background: #f9fafb;
    padding: 1rem;
    border-radius: 8px;
  }
  
  .task-column h2 {
    font-size: 1.125rem;
    font-weight: 600;
    margin-bottom: 1rem;
  }
  
  .task-card {
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 6px;
    padding: 1rem;
    margin-bottom: 0.75rem;
    transition: box-shadow 0.2s;
  }
  
  .task-card:hover {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
  
  .task-card.optimistic {
    opacity: 0.6;
  }
  
  .task-card.done {
    opacity: 0.7;
  }
  
  .task-header {
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    margin-bottom: 0.5rem;
  }
  
  .task-header h3 {
    flex: 1;
    font-size: 1rem;
    font-weight: 500;
    margin: 0;
  }
  
  .task-description {
    color: #6b7280;
    font-size: 0.875rem;
    margin: 0.5rem 0;
  }
  
  .task-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 0.75rem;
  }
  
  .priority {
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: uppercase;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
  }
  
  .priority-low {
    background: #dbeafe;
    color: #1e40af;
  }
  
  .priority-medium {
    background: #fef3c7;
    color: #92400e;
  }
  
  .priority-high {
    background: #fee2e2;
    color: #991b1b;
  }
  
  button {
    font-size: 0.875rem;
    color: #ef4444;
    background: none;
    border: none;
    cursor: pointer;
  }
  
  button:hover {
    text-decoration: underline;
  }
</style>
```

### Task Form

**src/lib/components/TaskForm.svelte**
```svelte
<script lang="ts">
  import { Effect } from "effect"
  import { TaskService } from "$services/TaskService"
  import { AppRuntime } from "$effects/runtime"
  import type { CreateTaskInput, TaskPriority } from "$domain/Task"
  
  let { projectId, oncreate } = $props<{
    projectId: string
    oncreate?: () => void
  }>()
  
  let title = $state("")
  let description = $state("")
  let priority = $state<TaskPriority>("medium")
  let submitting = $state(false)
  let error = $state<string | null>(null)
  
  async function handleSubmit() {
    if (!title.trim()) return
    
    submitting = true
    error = null
    
    const input: CreateTaskInput = {
      title,
      description: description || undefined,
      priority,
      projectId
    }
    
    const result = await Effect.runPromiseExit(
      Effect.gen(function* () {
        const service = yield* TaskService
        return yield* service.createTask(input)
      }),
      { runtime: AppRuntime }
    )
    
    if (result._tag === "Success") {
      // Reset form
      title = ""
      description = ""
      priority = "medium"
      oncreate?.()
    } else {
      error = "Failed to create task"
    }
    
    submitting = false
  }
</script>

<form onsubmit={(e) => { e.preventDefault(); handleSubmit() }}>
  <h2>New Task</h2>
  
  {#if error}
    <div class="error">{error}</div>
  {/if}
  
  <div class="field">
    <label for="title">Title *</label>
    <input
      id="title"
      type="text"
      bind:value={title}
      placeholder="Enter task title"
      required
    />
  </div>
  
  <div class="field">
    <label for="description">Description</label>
    <textarea
      id="description"
      bind:value={description}
      placeholder="Enter task description"
      rows="3"
    ></textarea>
  </div>
  
  <div class="field">
    <label for="priority">Priority</label>
    <select id="priority" bind:value={priority}>
      <option value="low">Low</option>
      <option value="medium">Medium</option>
      <option value="high">High</option>
    </select>
  </div>
  
  <button type="submit" disabled={!title.trim() || submitting}>
    {submitting ? "Creating..." : "Create Task"}
  </button>
</form>

<style>
  form {
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    padding: 1.5rem;
  }
  
  h2 {
    font-size: 1.25rem;
    font-weight: 600;
    margin-bottom: 1.5rem;
  }
  
  .error {
    background: #fee;
    color: #c00;
    padding: 0.75rem;
    border-radius: 4px;
    margin-bottom: 1rem;
    font-size: 0.875rem;
  }
  
  .field {
    margin-bottom: 1rem;
  }
  
  label {
    display: block;
    font-weight: 500;
    margin-bottom: 0.5rem;
    font-size: 0.875rem;
  }
  
  input,
  textarea,
  select {
    width: 100%;
    padding: 0.5rem 0.75rem;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    font-size: 0.875rem;
  }
  
  input:focus,
  textarea:focus,
  select:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }
  
  button {
    width: 100%;
    padding: 0.75rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.2s;
  }
  
  button:hover:not(:disabled) {
    background: #2563eb;
  }
  
  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
```

---

[↑ Back to Top](#table-of-contents)

## Routes

### Main Page

**src/routes/+page.svelte**
```svelte
<script lang="ts">
  import { Effect, Ref } from "effect"
  import { TaskService } from "$services/TaskService"
  import { ProjectService } from "$services/ProjectService"
  import { AppRuntime } from "$effects/runtime"
  import TaskList from "$lib/components/TaskList.svelte"
  import TaskForm from "$lib/components/TaskForm.svelte"
  import type { Project } from "$domain/Project"
  
  let projects = $state<Project[]>([])
  let selectedProjectId = $state<string | null>(null)
  let showForm = $state(false)
  let loading = $state(true)
  
  // Load projects
  $effect(() => {
    Effect.runPromise(
      Effect.gen(function* () {
        const service = yield* ProjectService
        return yield* service.listProjects()
      }),
      { runtime: AppRuntime }
    ).then((loadedProjects) => {
      projects = loadedProjects
      if (loadedProjects.length > 0) {
        selectedProjectId = loadedProjects[0].id
      }
      loading = false
    })
  })
  
  let selectedProject = $derived(
    projects.find((p) => p.id === selectedProjectId)
  )
</script>

<div class="container">
  <header>
    <h1>TaskMaster</h1>
    <button onclick={() => (showForm = !showForm)}>
      {showForm ? "Cancel" : "New Task"}
    </button>
  </header>
  
  {#if loading}
    <div class="loading">Loading...</div>
  {:else}
    <div class="project-selector">
      <label>Project:</label>
      <select bind:value={selectedProjectId}>
        {#each projects as project (project.id)}
          <option value={project.id}>{project.name}</option>
        {/each}
      </select>
    </div>
    
    {#if showForm && selectedProjectId}
      <TaskForm
        projectId={selectedProjectId}
        oncreate={() => (showForm = false)}
      />
    {/if}
    
    {#if selectedProjectId}
      <TaskList projectId={selectedProjectId} />
    {/if}
  {/if}
</div>

<style>
  .container {
    max-width: 1400px;
    margin: 0 auto;
    padding: 2rem;
  }
  
  header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
  }
  
  h1 {
    font-size: 2rem;
    font-weight: 700;
    color: #111827;
  }
  
  header button {
    padding: 0.5rem 1rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 6px;
    font-weight: 500;
    cursor: pointer;
  }
  
  .project-selector {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-bottom: 2rem;
  }
  
  .project-selector label {
    font-weight: 500;
  }
  
  .project-selector select {
    padding: 0.5rem 1rem;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    background: white;
  }
  
  .loading {
    text-align: center;
    padding: 3rem;
    color: #6b7280;
  }
</style>
```

---

[↑ Back to Top](#table-of-contents)

## Running the Project

### Development

```bash
# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

### Environment Variables

Create `.env`:

```bash
VITE_API_URL=http://localhost:3000/api
VITE_WS_URL=ws://localhost:3000/ws
```

### Mock API Server

For development, you can create a simple mock API server or use the built-in SvelteKit API routes with in-memory storage.

---

[↑ Back to Top](#table-of-contents)

## Key Patterns Demonstrated

1. **Effect Service Layer**: Complete service architecture with Context and Layers
2. **Svelte 5 Runes**: Modern `$state`, `$derived`, `$effect` patterns
3. **Optimistic Updates**: Immediate UI feedback with rollback on error
4. **Type Safety**: Full TypeScript with Effect Schema validation
5. **Error Handling**: Typed errors with discriminated unions
6. **Real-time Updates**: WebSocket integration with Effect Streams
7. **Composable Services**: Clean separation of concerns
8. **Reusable Patterns**: Custom runes for Effect integration

---

This complete project demonstrates all the concepts from the course in a real-world, production-ready application using the latest Svelte 5 features with Effect-TS!
