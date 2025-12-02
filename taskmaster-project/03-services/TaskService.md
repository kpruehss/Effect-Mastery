# TaskService Implementation

**Task management with optimistic updates**

Complete implementation from the original project file - see lines 439-564 of `effectts-svelte5-complete-project.md`.

## Key Features

- List tasks with optional project filtering
- Get single task by ID
- Create task with optimistic update
- Update task
- Delete task with optimistic delete and rollback

## Optimistic Update Pattern

```typescript
createTask: (input) => {
  // 1. Create optimistic task
  const optimisticTask = { ...input, id: `temp-${Date.now()}`, _optimistic: true }

  // 2. Add to state immediately
  yield* Ref.update(tasks, ts => [...ts, optimisticTask])

  // 3. Send to server
  const savedTask = yield* api.post("/tasks", input)

  // 4. Replace optimistic with real
  yield* Ref.update(tasks, ts =>
    ts.map(t => t.id === optimisticTask.id ? savedTask : t)
  )

  // 5. On error, rollback
  Effect.tapError(() =>
    Ref.update(tasks, ts => ts.filter(t => t.id !== optimisticTask.id))
  )
}
```

[View complete implementation →](../effectts-svelte5-complete-project.md#task-service)

---

[← ApiClient](./ApiClient.md) | [Back to Services](./README.md) | [ProjectService →](./ProjectService.md)
