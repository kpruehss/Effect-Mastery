# Chapter 6: Routes & Pages

**SvelteKit routes with server-side Effects**

In this chapter, we'll create the application routes, pages, and API endpoints using SvelteKit with Effect integration.

---

## Table of Contents

1. [Route Structure](#route-structure)
2. [Main Dashboard](#main-dashboard)
3. [SvelteKit Load Functions](#sveltekit-load-functions)
4. [Server Actions](#server-actions)
5. [API Routes](#api-routes)
6. [Next Steps](#next-steps)

---

## Route Structure

```
src/routes/
├── +layout.svelte          # Root layout with CSS
├── +page.svelte            # Dashboard (main page)
├── tasks/
│   ├── +page.svelte        # Task list page
│   └── [id]/
│       ├── +page.svelte    # Task detail page
│       └── +page.ts        # Load function
├── projects/
│   ├── +page.svelte        # Project list
│   └── [id]/
│       └── +page.svelte    # Project detail
└── api/
    ├── tasks/
    │   ├── +server.ts      # Task API endpoints
    │   └── [id]/
    │       └── +server.ts  # Single task endpoints
    └── projects/
        └── +server.ts      # Project API endpoints
```

---

## Main Dashboard

The main page that brings everything together.

### Implementation

Complete implementation from original project - see lines 1298-1423 of `effectts-svelte5-complete-project.md`.

Create `src/routes/+page.svelte`:

```svelte
<script lang="ts">
  import { Effect } from "effect"
  import { ProjectService } from "$services/ProjectService"
  import { AppRuntime } from "$effects/runtime"
  import TaskList from "$components/TaskList.svelte"
  import TaskForm from "$components/TaskForm.svelte"

  let projects = $state([])
  let selectedProjectId = $state(null)
  let showForm = $state(false)
  let loading = $state(true)

  // Load projects
  $effect(() => {
    Effect.runPromise(
      pipe(
        ProjectService,
        Effect.flatMap((service) => service.listProjects())
      ),
      { runtime: AppRuntime }
    ).then((loadedProjects) => {
      projects = loadedProjects
      if (loadedProjects.length > 0) {
        selectedProjectId = loadedProjects[0].id
      }
      loading = false
    })
  })
</script>

<div class="container mx-auto p-4">
  <header class="flex justify-between items-center mb-6">
    <h1 class="text-3xl font-bold">TaskMaster</h1>
    <button
      class="btn btn-primary"
      onclick={() => (showForm = !showForm)}
    >
      {showForm ? 'Cancel' : 'New Task'}
    </button>
  </header>

  {#if loading}
    <p>Loading...</p>
  {:else}
    <!-- Project selector -->
    <div class="mb-4">
      <select bind:value={selectedProjectId} class="form-select">
        {#each projects as project}
          <option value={project.id}>{project.name}</option>
        {/each}
      </select>
    </div>

    <!-- Task form -->
    {#if showForm && selectedProjectId}
      <TaskForm
        projectId={selectedProjectId}
        oncreate={() => (showForm = false)}
      />
    {/if}

    <!-- Task list -->
    {#if selectedProjectId}
      <TaskList projectId={selectedProjectId} />
    {/if}
  {/if}
</div>
```

---

## SvelteKit Load Functions

### Server-Side Data Loading

```typescript
// src/routes/tasks/[id]/+page.ts
import { Effect } from "effect"
import { error } from "@sveltejs/kit"
import { AppRuntime } from "$effects/runtime"
import { TaskService } from "$services/TaskService"
import type { PageLoad } from "./$types"

export const load: PageLoad = async ({ params }) => {
  const result = await Effect.runPromiseExit(
    pipe(
      TaskService,
      Effect.flatMap((service) => service.getTask(params.id))
    ),
    { runtime: AppRuntime }
  )

  if (result._tag === "Success") {
    return { task: result.value }
  } else {
    throw error(404, "Task not found")
  }
}
```

### Using Loaded Data

```svelte
<!-- src/routes/tasks/[id]/+page.svelte -->
<script lang="ts">
  import type { PageData } from "./$types"

  let { data } = $props<{ data: PageData }>()
</script>

<div class="card p-6">
  <h1 class="text-2xl font-bold mb-4">{data.task.title}</h1>
  <p>{data.task.description}</p>

  <div class="mt-4">
    <span class="badge badge-{data.task.priority}">
      {data.task.priority}
    </span>
    <span class="badge badge-{data.task.status}">
      {data.task.status}
    </span>
  </div>
</div>
```

---

## Server Actions

### Form Actions with Effect

```typescript
// src/routes/tasks/+page.server.ts
import { Effect } from "effect"
import { fail } from "@sveltejs/kit"
import { AppRuntime } from "$effects/runtime"
import { TaskService } from "$services/TaskService"
import type { Actions } from "./$types"

export const actions = {
  create: async ({ request }) => {
    const formData = await request.formData()
    const title = formData.get("title") as string
    const priority = formData.get("priority") as TaskPriority
    const projectId = formData.get("projectId") as string

    const result = await Effect.runPromiseExit(
      pipe(
        TaskService,
        Effect.flatMap((service) =>
          service.createTask({
            title,
            priority,
            projectId
          })
        )
      ),
      { runtime: AppRuntime }
    )

    if (result._tag === "Success") {
      return { success: true, task: result.value }
    } else {
      return fail(400, {
        error: "Failed to create task",
        message: result.cause._tag
      })
    }
  },

  update: async ({ request, params }) => {
    const formData = await request.formData()
    const title = formData.get("title") as string | null

    const result = await Effect.runPromiseExit(
      pipe(
        TaskService,
        Effect.flatMap((service) =>
          service.updateTask(params.id, {
            title: title || undefined
          })
        )
      ),
      { runtime: AppRuntime }
    )

    if (result._tag === "Success") {
      return { success: true }
    } else {
      return fail(400, { error: "Failed to update task" })
    }
  }
} satisfies Actions
```

### Using Actions in Components

```svelte
<script lang="ts">
  import { enhance } from "$app/forms"
  import type { ActionData } from "./$types"

  let { form } = $props<{ form: ActionData }>()
</script>

<form method="POST" action="?/create" use:enhance>
  <input name="title" placeholder="Task title" required />
  <select name="priority" required>
    <option value="low">Low</option>
    <option value="medium">Medium</option>
    <option value="high">High</option>
  </select>
  <input type="hidden" name="projectId" value={projectId} />

  {#if form?.error}
    <p class="error">{form.message}</p>
  {/if}

  <button type="submit">Create Task</button>
</form>
```

---

## API Routes

### GET Endpoint

```typescript
// src/routes/api/tasks/+server.ts
import { Effect } from "effect"
import { json, error } from "@sveltejs/kit"
import { AppRuntime } from "$effects/runtime"
import { TaskService } from "$services/TaskService"
import type { RequestHandler } from "./$types"

export const GET: RequestHandler = async ({ url }) => {
  const projectId = url.searchParams.get("projectId")

  const result = await Effect.runPromiseExit(
    pipe(
      TaskService,
      Effect.flatMap((service) => service.listTasks(projectId || undefined))
    ),
    { runtime: AppRuntime }
  )

  if (result._tag === "Success") {
    return json(result.value)
  } else {
    throw error(500, "Failed to fetch tasks")
  }
}
```

### POST Endpoint

```typescript
export const POST: RequestHandler = async ({ request }) => {
  const body = await request.json()

  const result = await Effect.runPromiseExit(
    pipe(
      TaskService,
      Effect.flatMap((service) =>
        pipe(
          validateCreateTaskInput(body),
          Effect.flatMap((validated) => service.createTask(validated))
        )
      )
    ),
    { runtime: AppRuntime }
  )

  if (result._tag === "Success") {
    return json(result.value, { status: 201 })
  } else {
    throw error(400, "Invalid task data")
  }
}
```

### DELETE Endpoint

```typescript
// src/routes/api/tasks/[id]/+server.ts
export const DELETE: RequestHandler = async ({ params }) => {
  const result = await Effect.runPromiseExit(
    pipe(
      TaskService,
      Effect.flatMap((service) => service.deleteTask(params.id))
    ),
    { runtime: AppRuntime }
  )

  if (result._tag === "Success") {
    return new Response(null, { status: 204 })
  } else {
    throw error(404, "Task not found")
  }
}
```

---

## Route Checklist

Before moving to Chapter 7, verify:

- ✅ Main dashboard page working
- ✅ Load functions fetching data
- ✅ Server actions handling forms
- ✅ API routes responding
- ✅ Error handling in place
- ✅ TypeScript types generated

---

## What We Accomplished

In this chapter, we:

1. ✅ Created main dashboard route
2. ✅ Implemented SvelteKit load functions with Effect
3. ✅ Added server actions for forms
4. ✅ Built API routes for external access
5. ✅ Handled errors gracefully
6. ✅ Integrated all components

---

## Next Steps

With our routes complete, we'll move to [Chapter 7: Deployment](../07-deployment/README.md) where we'll:

- Configure environment variables
- Optimize for production
- Deploy to Vercel/Netlify
- Set up monitoring
- Configure analytics

[Continue to Chapter 7 →](../07-deployment/README.md)

---

[← Chapter 5](../05-components/README.md) | [Main](../README.md) | [Chapter 7 →](../07-deployment/README.md)
