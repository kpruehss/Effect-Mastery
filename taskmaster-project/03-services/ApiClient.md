# ApiClient Service Implementation

**HTTP request wrapper with Effect patterns**

---

## Complete Implementation

Create `src/lib/services/ApiClient.ts`:

```typescript
import { Effect, Context, Layer } from "effect";
import type { ApiError } from "$domain/errors";

// Service interface
export interface ApiClient {
  readonly get: <T>(url: string) => Effect.Effect<T, ApiError>;
  readonly post: <T, B>(url: string, body: B) => Effect.Effect<T, ApiError>;
  readonly put: <T, B>(url: string, body: B) => Effect.Effect<T, ApiError>;
  readonly delete: (url: string) => Effect.Effect<void, ApiError>;
}

// Context tag
export const ApiClient = Context.GenericTag<ApiClient>("ApiClient");

// Configuration
interface Config {
  readonly baseUrl: string;
  readonly timeout: number;
}

// Service implementation
const makeApiClient = (config: Config): ApiClient => ({
  get: <T>(url: string) =>
    Effect.tryPromise({
      try: async () => {
        const controller = new AbortController();
        const timeout = setTimeout(() => controller.abort(), config.timeout);

        const response = await fetch(`${config.baseUrl}${url}`, {
          signal: controller.signal,
          credentials: "include",
        });

        clearTimeout(timeout);

        if (!response.ok) {
          throw { status: response.status };
        }

        return response.json() as Promise<T>;
      },
      catch: (error) =>
        typeof error === "object" && error !== null && "status" in error
          ? { _tag: "InvalidResponse" as const, status: error.status as number }
          : { _tag: "NetworkError" as const, cause: error },
    }),

  post: <T, B>(url: string, body: B) =>
    Effect.tryPromise({
      try: async () => {
        const response = await fetch(`${config.baseUrl}${url}`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          credentials: "include",
          body: JSON.stringify(body),
        });

        if (!response.ok) {
          throw { status: response.status };
        }

        return response.json() as Promise<T>;
      },
      catch: (error) =>
        typeof error === "object" && error !== null && "status" in error
          ? { _tag: "InvalidResponse" as const, status: error.status as number }
          : { _tag: "NetworkError" as const, cause: error },
    }),

  put: <T, B>(url: string, body: B) =>
    Effect.tryPromise({
      try: async () => {
        const response = await fetch(`${config.baseUrl}${url}`, {
          method: "PUT",
          headers: { "Content-Type": "application/json" },
          credentials: "include",
          body: JSON.stringify(body),
        });

        if (!response.ok) {
          throw { status: response.status };
        }

        return response.json() as Promise<T>;
      },
      catch: (error) =>
        typeof error === "object" && error !== null && "status" in error
          ? { _tag: "InvalidResponse" as const, status: error.status as number }
          : { _tag: "NetworkError" as const, cause: error },
    }),

  delete: (url: string) =>
    Effect.tryPromise({
      try: async () => {
        const response = await fetch(`${config.baseUrl}${url}`, {
          method: "DELETE",
          credentials: "include",
        });

        if (!response.ok) {
          throw { status: response.status };
        }
      },
      catch: (error) =>
        typeof error === "object" && error !== null && "status" in error
          ? { _tag: "InvalidResponse" as const, status: error.status as number }
          : { _tag: "NetworkError" as const, cause: error },
    }),
});

// Live layer
export const ApiClientLive = Layer.succeed(
  ApiClient,
  makeApiClient({
    baseUrl: import.meta.env.VITE_API_URL || "/api",
    timeout: 30000,
  }),
);
```

---

## Usage Examples

### Basic GET Request

```typescript
import { Effect } from "effect";
import { ApiClient } from "$services/ApiClient";

const fetchUsers = Effect.gen(function* () {
  const api = yield* ApiClient;
  const users = yield* api.get<User[]>("/users");
  return users;
});
```

### POST with Body

```typescript
const createUser = (input: CreateUserInput) =>
  Effect.gen(function* () {
    const api = yield* ApiClient;
    const user = yield* api.post<User, CreateUserInput>("/users", input);
    return user;
  });
```

### Error Handling

```typescript
const fetchUserSafe = (id: string) =>
  Effect.gen(function* () {
    const api = yield* ApiClient;

    return yield* pipe(
      api.get<User>(`/users/${id}`),
      Effect.catchAll((error) => {
        if (error._tag === "InvalidResponse" && error.status === 404) {
          return Effect.succeed(null);
        }
        return Effect.fail(error);
      }),
    );
  });
```

---

## Key Features

### 1. Timeout Protection

```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), config.timeout);
```

Prevents hanging requests.

### 2. Credential Management

```typescript
credentials: "include";
```

Automatically sends cookies for authentication.

### 3. Type-Safe Errors

```typescript
catch: (error) =>
  typeof error === "object" && error !== null && "status" in error
    ? { _tag: "InvalidResponse" as const, status: error.status as number }
    : { _tag: "NetworkError" as const, cause: error }
```

Structured error handling with discriminated unions.

### 4. JSON Handling

Automatic JSON parsing for responses.

---

## Testing

### Mock Implementation

```typescript
// src/lib/services/ApiClient.test.ts
import { Layer, Effect } from "effect";
import { ApiClient } from "./ApiClient";

export const TestApiClient = Layer.succeed(ApiClient, {
  get: <T>(_url: string) => {
    // Return test data
    return Effect.succeed({ id: "123", name: "Test" } as T);
  },

  post: <T, B>(_url: string, _body: B) => Effect.succeed({ success: true } as T),

  put: <T, B>(_url: string, _body: B) => Effect.succeed({ success: true } as T),

  delete: (_url: string) => Effect.void,
});
```

### Using in Tests (Example)

```typescript
import { Effect, Layer } from "effect";
import { MyService, MyServiceLive } from "./MyService";
import { TestApiClient } from "./ApiClient.test";

describe("MyService", () => {
  const TestLayer = MyServiceLive.pipe(Layer.provide(TestApiClient));

  it("fetches data", async () => {
    const program = Effect.gen(function* () {
      const service = yield* MyService;
      return yield* service.getData();
    });

    const result = await Effect.runPromise(program.pipe(Effect.provide(TestLayer)));

    expect(result).toBeDefined();
  });
});
```

---

## Advanced Patterns

### Retry on Network Error

```typescript
const fetchWithRetry = pipe(
  api.get("/data"),
  Effect.retry({
    while: (error) => error._tag === "NetworkError",
    times: 3,
  }),
);
```

### Request Deduplication

```typescript
const cache = new Map();

const fetchCached = (url: string) =>
  Effect.gen(function* () {
    if (cache.has(url)) {
      return cache.get(url);
    }

    const api = yield* ApiClient;
    const data = yield* api.get(url);
    cache.set(url, data);
    return data;
  });
```

### Request Queue

```typescript
import { Queue } from "effect";

const makeQueuedClient = Effect.gen(function* () {
  const queue = yield* Queue.bounded<Request>(10);

  // Process queue in background
  yield* Effect.forkDaemon(Stream.fromQueue(queue).pipe(Stream.mapEffect((req) => processRequest(req))));

  return {
    enqueue: (req: Request) => Queue.offer(queue, req),
  };
});
```

---

## Environment Configuration

### Different Base URLs

```typescript
// Development
export const ApiClientDev = Layer.succeed(
  ApiClient,
  makeApiClient({
    baseUrl: "http://localhost:3000/api",
    timeout: 30000,
  }),
);

// Production
export const ApiClientProd = Layer.succeed(
  ApiClient,
  makeApiClient({
    baseUrl: "https://api.taskmaster.com",
    timeout: 10000,
  }),
);

// Use appropriate layer
export const ApiClientLive = import.meta.env.MODE === "production" ? ApiClientProd : ApiClientDev;
```

---

## Summary

The ApiClient service provides:

- ✅ Centralized HTTP communication
- ✅ Timeout protection
- ✅ Type-safe error handling
- ✅ Credential management
- ✅ Easy mocking for tests
- ✅ Foundation for all services

---

[← Back to Services](./README.md) | [TaskService →](./TaskService.md)
