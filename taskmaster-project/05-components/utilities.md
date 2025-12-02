# Effect-Runes Utilities

**Reusable Effect-Svelte integration helpers**

Complete implementation from original project - see lines 754-810 of `effectts-svelte5-complete-project.md`.

---

## Overview

These utilities bridge Effect and Svelte 5 runes:

- `createEffectState` - Run Effect and track loading/success/error states
- `createRefreshableEffect` - Same as above, but with refresh capability

---

## Implementation

Create `src/lib/stores/effect-runes.svelte.ts`:

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

## Usage Examples

### Basic Effect State

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
  <p>Loading...</p>
{:else if tasksState.current.status === "error"}
  <p>Error: {tasksState.current.error._tag}</p>
{:else if tasksState.current.status === "success"}
  <ul>
    {#each tasksState.current.data as task}
      <li>{task.title}</li>
    {/each}
  </ul>
{/if}
```

### Refreshable Effect

```svelte
<script lang="ts">
  import { createRefreshableEffect } from "$stores/effect-runes.svelte"

  const statsState = createRefreshableEffect(() =>
    Effect.gen(function* () {
      const api = yield* ApiClient
      return yield* api.get("/stats")
    })
  )
</script>

<div>
  {#if statsState.current.status === "success"}
    <Stats data={statsState.current.data} />
    <button onclick={() => statsState.refresh()}>
      Refresh
    </button>
  {/if}
</div>
```

---

[← Back to Components](./README.md)
