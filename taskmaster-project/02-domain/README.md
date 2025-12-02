# Chapter 2: Domain Models & Types

**Building type-safe domain models with Effect Schema**

In this chapter, we'll define our core domain models using Effect Schema for validation, creating a solid foundation for our application's business logic.

---

## Table of Contents

1. [Domain-Driven Design Basics](#domain-driven-design-basics)
2. [Task Model](#task-model)
3. [Project Model](#project-model)
4. [User Model](#user-model)
5. [Error Types](#error-types)
6. [Testing Domain Models](#testing-domain-models)
7. [Next Steps](#next-steps)

---

## Domain-Driven Design Basics

### Why Domain Models?

Domain models represent the core business concepts in our application. They:

- Enforce business rules at the type level
- Provide validation boundaries
- Make impossible states unrepresentable
- Serve as documentation
- Enable safe refactoring

### Effect Schema Benefits

Effect Schema provides:

- **Runtime validation**: Ensure data integrity
- **Type inference**: TypeScript types derived from schemas
- **Composability**: Build complex schemas from simple ones
- **Error messages**: Clear validation error reporting

---

## Task Model

Our Task model is the heart of the application.

### Create Task.ts

Create `src/lib/domain/Task.ts`:

```typescript
import { Schema } from "@effect/schema"
import { Effect } from "effect"

// Task status enum
export const TaskStatus = Schema.Literal("todo", "in_progress", "done")
export type TaskStatus = Schema.Schema.Type<typeof TaskStatus>

// Task priority enum
export const TaskPriority = Schema.Literal("low", "medium", "high")
export type TaskPriority = Schema.Schema.Type<typeof TaskPriority>

// Complete Task schema
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
```

### Understanding the Task Schema

**Required fields:**
- `id`: Unique identifier
- `title`: Task name
- `status`: Current state (todo, in_progress, done)
- `priority`: Urgency level
- `projectId`: Parent project reference
- `createdAt`, `updatedAt`: Timestamps

**Optional fields:**
- `description`: Detailed task information
- `assigneeId`: Who is responsible
- `dueDate`: When it should be completed
- `_optimistic`: Flag for optimistic UI updates

### Create Task Input Schema

Add to `Task.ts`:

```typescript
// Create task input - what users provide
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
```

**Validation rules:**
- Title: 1-200 characters
- Description: Optional
- Status: Defaults to "todo" if not provided
- Priority: Required
- ProjectId: Required
- AssigneeId: Optional
- DueDate: Optional

### Update Task Input Schema

Add to `Task.ts`:

```typescript
// Update task input - all fields optional
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
```

### Validation Functions

Add to `Task.ts`:

```typescript
// Validation helpers
export const validateTask = Schema.decode(Task)
export const validateCreateTaskInput = Schema.decode(CreateTaskInput)
export const validateUpdateTaskInput = Schema.decode(UpdateTaskInput)
```

**Usage example:**

```typescript
// Validating external data
const result = await Effect.runPromise(
  validateTask(untrustedData)
)
```

---

## Project Model

Projects organize tasks into groups.

### Create Project.ts

Create `src/lib/domain/Project.ts`:

```typescript
import { Schema } from "@effect/schema"

// Project schema
export const Project = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  description: Schema.optional(Schema.String),
  color: Schema.String,  // Hex color for UI
  ownerId: Schema.String,
  createdAt: Schema.Date,
  updatedAt: Schema.Date
})

export type Project = Schema.Schema.Type<typeof Project>

// Create project input
export const CreateProjectInput = Schema.Struct({
  name: Schema.String.pipe(
    Schema.minLength(1, { message: () => "Name is required" }),
    Schema.maxLength(100, { message: () => "Name too long" })
  ),
  description: Schema.optional(Schema.String),
  color: Schema.String  // Could add hex color validation
})

export type CreateProjectInput = Schema.Schema.Type<typeof CreateProjectInput>

// Validation helpers
export const validateProject = Schema.decode(Project)
export const validateCreateProjectInput = Schema.decode(CreateProjectInput)
```

### Optional: Add Color Validation

For stricter validation, add hex color validation:

```typescript
// Hex color validator
const HexColor = Schema.String.pipe(
  Schema.pattern(/^#[0-9A-Fa-f]{6}$/)
)

// Use in Project schema
export const CreateProjectInput = Schema.Struct({
  name: Schema.String.pipe(
    Schema.minLength(1, { message: () => "Name is required" }),
    Schema.maxLength(100, { message: () => "Name too long" })
  ),
  description: Schema.optional(Schema.String),
  color: HexColor  // Now validates hex format
})
```

---

## User Model

User information for authentication and assignment.

### Create User.ts

Create `src/lib/domain/User.ts`:

```typescript
import { Schema } from "@effect/schema"

// User schema
export const User = Schema.Struct({
  id: Schema.String,
  email: Schema.String,
  name: Schema.String,
  avatar: Schema.optional(Schema.String),  // URL to avatar image
  createdAt: Schema.Date
})

export type User = Schema.Schema.Type<typeof User>

// Validation helper
export const validateUser = Schema.decode(User)
```

### Optional: Add Email Validation

For stricter validation:

```typescript
// Email validator
const Email = Schema.String.pipe(
  Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
)

export const User = Schema.Struct({
  id: Schema.String,
  email: Email,  // Now validates email format
  name: Schema.String,
  avatar: Schema.optional(Schema.String),
  createdAt: Schema.Date
})
```

---

## Error Types

Define domain-specific error types using discriminated unions.

### Create errors.ts

Create `src/lib/domain/errors.ts`:

```typescript
// Task-related errors
export type TaskError =
  | { _tag: "TaskNotFound"; id: string }
  | { _tag: "InvalidTaskData"; errors: unknown }
  | { _tag: "Unauthorized" }

// Project-related errors
export type ProjectError =
  | { _tag: "ProjectNotFound"; id: string }
  | { _tag: "InvalidProjectData"; errors: unknown }
  | { _tag: "Unauthorized" }

// API-related errors
export type ApiError =
  | { _tag: "NetworkError"; cause: unknown }
  | { _tag: "InvalidResponse"; status: number }
  | { _tag: "Timeout" }

// Authentication errors
export type AuthError =
  | { _tag: "InvalidCredentials" }
  | { _tag: "SessionExpired" }
  | { _tag: "Unauthorized" }
```

### Why Discriminated Unions?

Discriminated unions with `_tag` enable:

1. **Exhaustive matching**: TypeScript ensures all cases are handled
2. **Type narrowing**: Access error-specific fields safely
3. **Better error messages**: Clear error categorization

**Usage example:**

```typescript
const handleError = (error: TaskError) => {
  switch (error._tag) {
    case "TaskNotFound":
      return `Task ${error.id} not found`
    case "InvalidTaskData":
      return "Invalid task data"
    case "Unauthorized":
      return "You don't have permission"
  }
}
```

---

## Testing Domain Models

Create basic tests to verify validation.

### Create Task.test.ts

Create `src/lib/domain/Task.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { Effect } from 'effect'
import { validateCreateTaskInput, type CreateTaskInput } from './Task'

describe('Task Domain', () => {
  it('validates valid task input', async () => {
    const input: CreateTaskInput = {
      title: 'Test Task',
      priority: 'medium',
      projectId: 'project-123'
    }

    const result = await Effect.runPromise(
      validateCreateTaskInput(input)
    )

    expect(result.title).toBe('Test Task')
    expect(result.priority).toBe('medium')
  })

  it('rejects empty title', async () => {
    const input = {
      title: '',
      priority: 'medium',
      projectId: 'project-123'
    }

    await expect(
      Effect.runPromise(validateCreateTaskInput(input))
    ).rejects.toThrow()
  })

  it('rejects title over 200 characters', async () => {
    const input = {
      title: 'a'.repeat(201),
      priority: 'medium',
      projectId: 'project-123'
    }

    await expect(
      Effect.runPromise(validateCreateTaskInput(input))
    ).rejects.toThrow()
  })
})
```

### Run Tests

```bash
npm test
```

All tests should pass!

---

## Domain Layer Checklist

Before moving to Chapter 3, verify:

- ✅ Task model with status and priority enums
- ✅ Task validation schemas (create, update)
- ✅ Project model with validation
- ✅ User model
- ✅ Error types defined
- ✅ Validation functions exported
- ✅ Tests passing

---

## Best Practices

### 1. Keep Domain Pure

Domain models should not depend on:
- UI frameworks
- Database libraries
- HTTP clients
- External services

They represent pure business logic.

### 2. Validate at Boundaries

Use validation functions when:
- Receiving data from API
- Processing user input
- Loading from database

### 3. Use Descriptive Error Types

Instead of generic strings, use specific error types:

```typescript
// ❌ Bad
throw new Error("not found")

// ✅ Good
return Effect.fail({ _tag: "TaskNotFound", id: taskId })
```

### 4. Make Illegal States Unrepresentable

Use types to prevent invalid states:

```typescript
// ✅ Good - Can't have invalid status
type TaskStatus = "todo" | "in_progress" | "done"

// ❌ Bad - Any string allowed
type TaskStatus = string
```

---

## What We Accomplished

In this chapter, we:

1. ✅ Created Task model with Effect Schema
2. ✅ Defined validation rules and constraints
3. ✅ Built Project and User models
4. ✅ Created domain error types
5. ✅ Wrote validation functions
6. ✅ Added basic tests

---

## Next Steps

With our domain models defined, we'll move to [Chapter 3: Service Layer](../03-services/README.md) where we'll:

- Build the API Client service
- Create TaskService with optimistic updates
- Implement ProjectService
- Add WebSocket service for real-time updates
- Handle offline scenarios

[Continue to Chapter 3 →](../03-services/README.md)

---

[← Chapter 1](../01-setup/README.md) | [Main](../README.md) | [Chapter 3 →](../03-services/README.md)
