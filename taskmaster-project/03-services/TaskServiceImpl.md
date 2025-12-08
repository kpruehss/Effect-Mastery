### Task Service

**src/lib/services/TaskService.ts**

```typescript
import { Effect, Context, Layer, pipe, Ref } from "effect";
import { ApiClient } from "./ApiClient";
import { validateTask, validateCreateTaskInput } from "$domain/Task";
import type { Task, CreateTaskInput, UpdateTaskInput } from "$domain/Task";
import type { TaskError } from "$domain/errors";

export interface TaskService {
  readonly tasks: Ref.Ref<Task[]>;
  readonly listTasks: (projectId?: string) => Effect.Effect<Task[], TaskError>;
  readonly getTask: (id: string) => Effect.Effect<Task, TaskError>;
  readonly createTask: (input: CreateTaskInput) => Effect.Effect<Task, TaskError>;
  readonly updateTask: (id: string, input: UpdateTaskInput) => Effect.Effect<Task, TaskError>;
  readonly deleteTask: (id: string) => Effect.Effect<void, TaskError>;
}

export class TaskService extends Context.Tag("TaskService")<
  TaskService,
  TaskService
>() {}

export const TaskServiceLive = Layer.effect(
  TaskService,
  pipe(
    Effect.all([ApiClient, Ref.make<Task[]>([])]),
    Effect.map(([api, tasks]) => ({
      tasks,

      listTasks: (projectId?: string) =>
        pipe(
          api.get<unknown[]>(projectId ? `/tasks?projectId=${projectId}` : "/tasks"),
          Effect.flatMap((data) => Effect.all(data.map(validateTask), { concurrency: "unbounded" })),
          Effect.tap((loadedTasks) => Ref.set(tasks, loadedTasks)),
          Effect.mapError(() => ({ _tag: "TaskNotFound" as const, id: "all" })),
        ),

      getTask: (id) =>
        pipe(
          api.get<unknown>(`/tasks/${id}`),
          Effect.flatMap(validateTask),
          Effect.mapError(() => ({ _tag: "TaskNotFound" as const, id })),
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
          _optimistic: true,
        };

        return pipe(
          // Add optimistically
          Ref.update(tasks, (ts) => [...ts, optimisticTask]),
          Effect.flatMap(() =>
            pipe(
              validateCreateTaskInput(input),
              Effect.flatMap((validated) => api.post<unknown, CreateTaskInput>("/tasks", validated)),
              Effect.flatMap(validateTask),
            ),
          ),
          Effect.tap((savedTask) =>
            // Replace optimistic with real
            Ref.update(tasks, (ts) => ts.map((t) => (t.id === optimisticTask.id ? savedTask : t))),
          ),
          Effect.tapError(() =>
            // Rollback on error
            Ref.update(tasks, (ts) => ts.filter((t) => t.id !== optimisticTask.id)),
          ),
          Effect.mapError(() => ({
            _tag: "InvalidTaskData" as const,
            errors: "Failed to create task",
          })),
        );
      },

      updateTask: (id, input) =>
        pipe(
          api.put<unknown, UpdateTaskInput>(`/tasks/${id}`, input),
          Effect.flatMap(validateTask),
          Effect.tap((updatedTask) => Ref.update(tasks, (ts) => ts.map((t) => (t.id === id ? updatedTask : t)))),
          Effect.mapError(() => ({ _tag: "TaskNotFound" as const, id })),
        ),

      deleteTask: (id) =>
        pipe(
          Ref.get(tasks),
          Effect.flatMap((ts) => {
            const task = ts.find((t) => t.id === id);
            if (!task) {
              return Effect.fail({ _tag: "TaskNotFound" as const, id });
            }

            return pipe(
              // Remove optimistically
              Ref.update(tasks, (ts) => ts.filter((t) => t.id !== id)),
              Effect.flatMap(() => api.delete(`/tasks/${id}`)),
              Effect.tapError(() =>
                // Rollback on error
                Ref.update(tasks, (ts) => [...ts, task]),
              ),
            );
          }),
        ),
    }))
  )
);
```
