# Chapter 5: Components & UI

**Building reactive Svelte 5 components with Effect integration**

In this chapter, we'll create reusable utilities for Effect-Svelte integration and build the UI components for our application.

---

## Table of Contents

1. [Effect-Runes Utilities](./utilities.md)
2. [TaskList Component](./TaskList.md)
3. [TaskForm Component](./TaskForm.md)
4. [Component Patterns](#component-patterns)
5. [Next Steps](#next-steps)

---

## Chapter Overview

We'll build:

1. **Reusable utilities** for Effect-Svelte integration
2. **TaskList** component with real-time updates
3. **TaskForm** with validation
4. **Dashboard** with analytics

---

## Component Architecture

### Data Flow

```
┌──────────────────────┐
│  Svelte Component    │
│                      │
│  ┌────────────────┐  │
│  │  $state        │  │ ← Local UI state
│  └────────────────┘  │
│         ↕             │
│  ┌────────────────┐  │
│  │  Effect Runes  │  │ ← Effect integration
│  └────────────────┘  │
│         ↕             │
│  ┌────────────────┐  │
│  │  AppRuntime    │  │ ← Execute Effects
│  └────────────────┘  │
│         ↕             │
│  ┌────────────────┐  │
│  │  Services      │  │ ← Business logic
│  └────────────────┘  │
└──────────────────────┘
```

---

## Quick Start

### 1. Create Effect Utilities

First, create reusable Effect-Svelte helpers:

[→ View utilities documentation](./utilities.md)

### 2. Build TaskList

Create the task list with drag-and-drop:

[→ View TaskList documentation](./TaskList.md)

### 3. Build TaskForm

Create form with validation:

[→ View TaskForm documentation](./TaskForm.md)

---

## Component Patterns

### Pattern 1: Loading States

```svelte
<script lang="ts">
  import { createEffectState } from "$stores/effect-runes.svelte"
  import { TaskService } from "$services/TaskService"

  const tasksState = createEffectState(
    Effect.gen(function* () {
      const service = yield* TaskService
      return yield* service.listTasks()
    })
  )
</script>

{#if tasksState.current.status === "loading"}
  <LoadingSpinner />
{:else if tasksState.current.status === "error"}
  <ErrorMessage error={tasksState.current.error} />
{:else if tasksState.current.status === "success"}
  <TaskList tasks={tasksState.current.data} />
{/if}
```

### Pattern 2: Optimistic Updates

```svelte
<script lang="ts">
  async function handleCreate(input: CreateTaskInput) {
    // UI updates immediately
    const result = await Effect.runPromiseExit(
      Effect.gen(function* () {
        const service = yield* TaskService
        return yield* service.createTask(input)
      }),
      { runtime: AppRuntime }
    )

    // Handle success/failure
    if (result._tag === "Success") {
      // Task already in list (optimistic)
    } else {
      // Show error, task removed (rollback)
      showError(result.cause)
    }
  }
</script>
```

### Pattern 3: Real-Time Updates

```svelte
<script lang="ts">
  import { WebSocketService } from "$services/WebSocketService"

  $effect(() => {
    const program = Effect.gen(function* () {
      const ws = yield* WebSocketService

      yield* Stream.runForEach(
        ws.messages,
        (message) => Effect.sync(() => {
          handleRealtimeUpdate(message)
        })
      )
    })

    const fiber = Effect.runFork(program, { runtime: AppRuntime })

    return () => {
      Effect.runFork(
        fiber.pipe(Effect.flatMap(f => f.interrupt))
      )
    }
  })
</script>
```

### Pattern 4: Derived State

```svelte
<script lang="ts">
  let tasks = $state<Task[]>([])
  let filter = $state<TaskStatus | null>(null)

  // Derived filtered tasks
  let filteredTasks = $derived(
    filter
      ? tasks.filter(t => t.status === filter)
      : tasks
  )

  // Derived statistics
  let stats = $derived({
    total: tasks.length,
    completed: tasks.filter(t => t.status === "done").length,
    inProgress: tasks.filter(t => t.status === "in_progress").length
  })
</script>

<div>
  <Stats {...stats} />
  <TaskList tasks={filteredTasks} />
</div>
```

---

## Styling with Tailwind

### Component Classes

```svelte
<!-- Card -->
<div class="bg-white border border-gray-200 rounded-lg shadow-sm p-4">
  Content
</div>

<!-- Button -->
<button class="btn btn-primary">
  Click me
</button>

<!-- Input -->
<input
  class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary-500"
  type="text"
/>

<!-- Badge -->
<span class="px-2 py-1 text-xs font-medium bg-blue-100 text-blue-800 rounded">
  Badge
</span>
```

---

## Component Checklist

Before moving to Chapter 6, verify:

- ✅ Effect-runes utilities created
- ✅ TaskList component working
- ✅ TaskForm with validation
- ✅ Loading states handled
- ✅ Error states displayed
- ✅ Optimistic updates working

---

## Component Files

### Core Utilities

- [effect-runes.svelte.ts](./utilities.md) - Effect-Svelte integration helpers

### Components

- [TaskList.svelte](./TaskList.md) - Task list with kanban view
- [TaskForm.svelte](./TaskForm.md) - Task creation/edit form
- [ProjectSelector.svelte](#) - Project dropdown
- [Dashboard.svelte](#) - Analytics dashboard

---

## Testing Components

### Component Tests

```typescript
import { render, screen } from '@testing-library/svelte'
import { Effect } from 'effect'
import TaskList from './TaskList.svelte'
import { TestLayer } from '$effects/test-layers'

describe('TaskList', () => {
  it('renders tasks', async () => {
    render(TaskList, {
      props: { projectId: 'test-123' }
    })

    // Wait for loading
    await screen.findByText('Test Task')

    expect(screen.getByText('Test Task')).toBeInTheDocument()
  })
})
```

---

## What We Accomplished

In this chapter, we:

1. ✅ Created Effect-Svelte utilities
2. ✅ Built TaskList component
3. ✅ Implemented TaskForm with validation
4. ✅ Handled loading and error states
5. ✅ Implemented optimistic updates
6. ✅ Added real-time update handling

---

## Next Steps

With our components built, we'll move to [Chapter 6: Routes & Pages](../06-routes/README.md) where we'll:

- Create main dashboard route
- Build task detail pages
- Implement SvelteKit load functions
- Add server actions
- Create API routes

[Continue to Chapter 6 →](../06-routes/README.md)

---

[← Chapter 4](../04-layers/README.md) | [Main](../README.md) | [Chapter 6 →](../06-routes/README.md)
