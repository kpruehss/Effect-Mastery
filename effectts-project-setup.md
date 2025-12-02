# EffectTS Project Setup Guide

**Complete setup instructions for React and Svelte projects**

---

## Table of Contents

1. [React Project Setup](#react-project-setup)
2. [Svelte Project Setup](#svelte-project-setup)
3. [Shared Configuration](#shared-configuration)
4. [Project Structure](#project-structure)
5. [Best Practices](#best-practices)
6. [CI/CD Integration](#cicd-integration)

---

## React Project Setup

### 1. Initialize Project

```bash
# Using Vite (recommended)
npm create vite@latest my-effect-app -- --template react-ts
cd my-effect-app

# Or using Create React App
npx create-react-app my-effect-app --template typescript
cd my-effect-app
```

### 2. Install Effect Dependencies

```bash
# Core Effect packages
npm install effect

# React integration
npm install @effect-rx/rx @effect-rx/rx-react

# Optional but recommended
npm install @effect/platform @effect/schema

# Development dependencies
npm install -D @effect/language-service
```

### 3. TypeScript Configuration

Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowJs": false,
    "checkJs": false,
    "jsx": "react-jsx",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "removeComments": false,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "skipLibCheck": true,
    "exactOptionalPropertyTypes": false,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### 4. Project Structure

```
my-effect-app/
├── src/
│   ├── effects/
│   │   ├── runtime.ts           # Effect runtime setup
│   │   └── layers.ts            # Service layers composition
│   ├── services/
│   │   ├── ApiClient.ts         # HTTP service
│   │   ├── Logger.ts            # Logging service
│   │   └── UserService.ts       # Domain services
│   ├── domain/
│   │   ├── User.ts              # Domain models
│   │   └── errors.ts            # Error types
│   ├── components/
│   │   ├── UserProfile.tsx
│   │   └── ...
│   ├── hooks/
│   │   ├── useEffectOnce.ts    # Custom Effect hooks
│   │   └── useEffectRx.ts
│   ├── App.tsx
│   └── main.tsx
├── package.json
├── tsconfig.json
└── vite.config.ts
```

### 5. Runtime Setup

Create `src/effects/runtime.ts`:

```typescript
import { Effect, Layer, Runtime } from "effect"
import { createContext, useContext, ReactNode } from "react"

// Import your layers
import { AppLayer } from "./layers"

// Create runtime with layers
const AppRuntime = Layer.toRuntime(AppLayer).pipe(Effect.runSync)

// React context for runtime
const RuntimeContext = createContext<typeof AppRuntime | null>(null)

export const RuntimeProvider = ({ children }: { children: ReactNode }) => {
  return (
    <RuntimeContext.Provider value={AppRuntime}>
      {children}
    </RuntimeContext.Provider>
  )
}

export const useRuntime = () => {
  const runtime = useContext(RuntimeContext)
  if (!runtime) {
    throw new Error("Runtime not provided. Wrap your app with RuntimeProvider.")
  }
  return runtime
}

export { AppRuntime }
```

Create `src/effects/layers.ts`:

```typescript
import { Layer } from "effect"
import { ApiClientLive } from "../services/ApiClient"
import { LoggerLive } from "../services/Logger"
import { UserServiceLive } from "../services/UserService"

// Compose all application layers
export const AppLayer = Layer.mergeAll(
  UserServiceLive
).pipe(
  Layer.provide(Layer.mergeAll(ApiClientLive, LoggerLive))
)
```

### 6. Example Service

Create `src/services/Logger.ts`:

```typescript
import { Effect, Context, Layer, Console } from "effect"

export interface Logger {
  readonly debug: (message: string) => Effect.Effect<void>
  readonly info: (message: string) => Effect.Effect<void>
  readonly warn: (message: string) => Effect.Effect<void>
  readonly error: (message: string) => Effect.Effect<void>
}

export const Logger = Context.GenericTag<Logger>("Logger")

const makeLogger = (): Logger => ({
  debug: (message) => Console.log(`[DEBUG] ${message}`),
  info: (message) => Console.log(`[INFO] ${message}`),
  warn: (message) => Console.warn(`[WARN] ${message}`),
  error: (message) => Console.error(`[ERROR] ${message}`)
})

export const LoggerLive = Layer.succeed(Logger, makeLogger())
```

### 7. App Setup

Update `src/main.tsx`:

```typescript
import React from "react"
import ReactDOM from "react-dom/client"
import App from "./App"
import { RuntimeProvider } from "./effects/runtime"
import "./index.css"

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <RuntimeProvider>
      <App />
    </RuntimeProvider>
  </React.StrictMode>
)
```

### 8. Example Component

Create `src/components/UserProfile.tsx`:

```typescript
import { Rx, useRx } from "@effect-rx/rx-react"
import { Effect, pipe } from "effect"
import { UserService } from "../services/UserService"

const userRx = (userId: string) =>
  Rx.make(() =>
    pipe(
      UserService,
      Effect.flatMap(service => service.getUser(userId))
    )
  )

export function UserProfile({ userId }: { userId: string }) {
  const user = useRx(userRx(userId))
  
  if (user._tag === "loading") {
    return <div>Loading...</div>
  }
  
  if (user._tag === "error") {
    return <div>Error: {JSON.stringify(user.error)}</div>
  }
  
  return (
    <div>
      <h1>{user.value.name}</h1>
      <p>{user.value.email}</p>
    </div>
  )
}
```

### 9. Testing Setup

Install testing dependencies:

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```

Update `vite.config.ts`:

```typescript
import { defineConfig } from "vite"
import react from "@vitejs/plugin-react"

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: "./src/test/setup.ts"
  }
})
```

Create `src/test/setup.ts`:

```typescript
import { expect, afterEach } from "vitest"
import { cleanup } from "@testing-library/react"
import * as matchers from "@testing-library/jest-dom/matchers"

expect.extend(matchers)

afterEach(() => {
  cleanup()
})
```

---

## Svelte Project Setup

### 1. Initialize Project

```bash
# Using SvelteKit (recommended)
npm create svelte@latest my-effect-app
cd my-effect-app
npm install

# Choose:
# - Skeleton project or SvelteKit demo app
# - Yes, using TypeScript syntax
# - Add ESLint, Prettier, Playwright, Vitest
```

### 2. Install Effect Dependencies

```bash
# Core Effect packages
npm install effect

# Optional but recommended
npm install @effect/platform @effect/schema

# Development dependencies
npm install -D @effect/language-service
```

### 3. TypeScript Configuration

Update `tsconfig.json`:

```json
{
  "extends": "./.svelte-kit/tsconfig.json",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowJs": true,
    "checkJs": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "paths": {
      "$lib": ["./src/lib"],
      "$lib/*": ["./src/lib/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.svelte"],
  "exclude": ["node_modules"]
}
```

### 4. Project Structure

```
my-effect-app/
├── src/
│   ├── lib/
│   │   ├── effects/
│   │   │   ├── runtime.ts
│   │   │   └── layers.ts
│   │   ├── services/
│   │   │   ├── ApiClient.ts
│   │   │   ├── Logger.ts
│   │   │   └── UserService.ts
│   │   ├── domain/
│   │   │   ├── User.ts
│   │   │   └── errors.ts
│   │   ├── stores/
│   │   │   ├── effect-store.ts
│   │   │   └── user-store.ts
│   │   └── components/
│   │       └── UserProfile.svelte
│   ├── routes/
│   │   ├── +layout.svelte
│   │   ├── +page.svelte
│   │   └── users/
│   │       └── [id]/
│   │           ├── +page.ts
│   │           └── +page.svelte
│   └── app.html
├── package.json
├── svelte.config.js
└── tsconfig.json
```

### 5. Effect Store Helper

Create `src/lib/stores/effect-store.ts`:

```typescript
import { writable, type Readable } from "svelte/store"
import { Effect, Runtime, Exit } from "effect"

export function effectStore<A, E>(
  effect: Effect.Effect<A, E, never>,
  runtime: Runtime.Runtime<never> = Runtime.defaultRuntime
): Readable<
  | { _tag: "loading" }
  | { _tag: "success"; data: A }
  | { _tag: "error"; error: E }
> {
  const store = writable<
    | { _tag: "loading" }
    | { _tag: "success"; data: A }
    | { _tag: "error"; error: E }
  >({ _tag: "loading" })
  
  Effect.runPromiseExit(effect, { runtime }).then(exit => {
    if (Exit.isSuccess(exit)) {
      store.set({ _tag: "success", data: exit.value })
    } else {
      store.set({ _tag: "error", error: Exit.unannotate(exit.cause) })
    }
  })
  
  return {
    subscribe: store.subscribe
  }
}

export function refreshableEffectStore<A, E>(
  effectFn: () => Effect.Effect<A, E, never>,
  runtime: Runtime.Runtime<never> = Runtime.defaultRuntime
) {
  const store = writable<
    | { _tag: "loading" }
    | { _tag: "success"; data: A }
    | { _tag: "error"; error: E }
  >({ _tag: "loading" })
  
  const load = () => {
    store.set({ _tag: "loading" })
    Effect.runPromiseExit(effectFn(), { runtime }).then(exit => {
      if (Exit.isSuccess(exit)) {
        store.set({ _tag: "success", data: exit.value })
      } else {
        store.set({ _tag: "error", error: Exit.unannotate(exit.cause) })
      }
    })
  }
  
  load() // Initial load
  
  return {
    subscribe: store.subscribe,
    refresh: load
  }
}
```

### 6. Runtime Setup

Create `src/lib/effects/runtime.ts`:

```typescript
import { Effect, Layer, Runtime } from "effect"
import { AppLayer } from "./layers"

export const AppRuntime = Layer.toRuntime(AppLayer).pipe(Effect.runSync)
```

Create `src/lib/effects/layers.ts`:

```typescript
import { Layer } from "effect"
import { ApiClientLive } from "../services/ApiClient"
import { LoggerLive } from "../services/Logger"
import { UserServiceLive } from "../services/UserService"

export const AppLayer = Layer.mergeAll(
  UserServiceLive
).pipe(
  Layer.provide(Layer.mergeAll(ApiClientLive, LoggerLive))
)
```

### 7. Example Component

Create `src/lib/components/UserProfile.svelte`:

```svelte
<script lang="ts">
  import { effectStore } from "$lib/stores/effect-store"
  import { fetchUser } from "$lib/services/UserService"
  
  export let userId: string
  
  $: userStore = effectStore(fetchUser(userId))
</script>

{#if $userStore._tag === "loading"}
  <div class="loading">
    <div class="spinner" />
    <p>Loading user...</p>
  </div>
{:else if $userStore._tag === "error"}
  <div class="error">
    <p>Failed to load user</p>
    <pre>{JSON.stringify($userStore.error, null, 2)}</pre>
  </div>
{:else}
  <div class="user-profile">
    <h1>{$userStore.data.name}</h1>
    <p>{$userStore.data.email}</p>
  </div>
{/if}

<style>
  .loading {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 2rem;
  }
  
  .spinner {
    width: 40px;
    height: 40px;
    border: 4px solid #f3f3f3;
    border-top: 4px solid #3498db;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  .error {
    color: red;
    padding: 1rem;
  }
  
  .user-profile {
    padding: 1rem;
  }
</style>
```

### 8. SvelteKit Page

Create `src/routes/users/[id]/+page.ts`:

```typescript
import { Effect } from "effect"
import { fetchUser } from "$lib/services/UserService"
import { error } from "@sveltejs/kit"
import type { PageLoad } from "./$types"

export const load: PageLoad = async ({ params }) => {
  const result = await Effect.runPromiseExit(fetchUser(params.id))
  
  if (result._tag === "Success") {
    return {
      user: result.value
    }
  } else {
    throw error(404, "User not found")
  }
}
```

Create `src/routes/users/[id]/+page.svelte`:

```svelte
<script lang="ts">
  import type { PageData } from "./$types"
  
  export let data: PageData
</script>

<div class="user-page">
  <h1>{data.user.name}</h1>
  <p>{data.user.email}</p>
</div>
```

### 9. Testing Setup

Create `src/test/setup.ts`:

```typescript
import { expect, afterEach } from "vitest"
import { cleanup } from "@testing-library/svelte"
import * as matchers from "@testing-library/jest-dom/matchers"

expect.extend(matchers)

afterEach(() => {
  cleanup()
})
```

Update `vite.config.ts`:

```typescript
import { sveltekit } from "@sveltejs/kit/vite"
import { defineConfig } from "vitest/config"

export default defineConfig({
  plugins: [sveltekit()],
  test: {
    include: ["src/**/*.{test,spec}.{js,ts}"],
    globals: true,
    environment: "jsdom",
    setupFiles: ["./src/test/setup.ts"]
  }
})
```

---

## Shared Configuration

### ESLint Configuration

Create `.eslintrc.cjs`:

```javascript
module.exports = {
  root: true,
  parser: "@typescript-eslint/parser",
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  plugins: ["@typescript-eslint"],
  parserOptions: {
    sourceType: "module",
    ecmaVersion: 2022
  },
  env: {
    browser: true,
    es2022: true,
    node: true
  },
  rules: {
    "@typescript-eslint/no-unused-vars": [
      "error",
      { argsIgnorePattern: "^_" }
    ],
    "@typescript-eslint/no-explicit-any": "warn"
  }
}
```

### Prettier Configuration

Create `.prettierrc`:

```json
{
  "semi": false,
  "singleQuote": false,
  "trailingComma": "none",
  "printWidth": 90,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "avoid"
}
```

### VS Code Settings

Create `.vscode/settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true
}
```

### VS Code Extensions

Create `.vscode/extensions.json`:

```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "effectful-tech.effect-vscode"
  ]
}
```

---

## Project Structure Best Practices

### Domain Layer

```typescript
// src/domain/User.ts
import { Schema } from "@effect/schema"

export const User = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String,
  role: Schema.Literal("admin", "user", "viewer")
})

export type User = Schema.Schema.Type<typeof User>

export const validateUser = Schema.decode(User)
```

### Error Types

```typescript
// src/domain/errors.ts
export type ApiError =
  | { _tag: "NetworkError"; cause: unknown }
  | { _tag: "InvalidResponse"; status: number }
  | { _tag: "Timeout" }

export type ValidationError =
  | { _tag: "Required"; field: string }
  | { _tag: "InvalidFormat"; field: string }

export type UserError =
  | { _tag: "UserNotFound"; id: string }
  | { _tag: "InvalidUser"; errors: ValidationError[] }
```

### Service Layer

```typescript
// src/services/UserService.ts
import { Effect, Context, Layer, pipe } from "effect"
import type { User } from "../domain/User"
import type { UserError } from "../domain/errors"
import { ApiClient } from "./ApiClient"

export interface UserService {
  readonly getUser: (id: string) => Effect.Effect<User, UserError>
  readonly listUsers: () => Effect.Effect<User[], UserError>
}

export const UserService = Context.GenericTag<UserService>("UserService")

const makeUserService = Effect.gen(function* () {
  const api = yield* ApiClient
  
  return UserService.of({
    getUser: (id) =>
      pipe(
        api.get<User>(`/users/${id}`),
        Effect.mapError(() => ({ _tag: "UserNotFound" as const, id }))
      ),
    
    listUsers: () =>
      pipe(
        api.get<User[]>("/users"),
        Effect.mapError(() => ({ _tag: "UserNotFound" as const, id: "all" }))
      )
  })
})

export const UserServiceLive = Layer.effect(UserService, makeUserService)
```

---

## Best Practices

### 1. Error Handling

```typescript
// Always use discriminated unions for errors
type DomainError = 
  | { _tag: "NotFound"; id: string }
  | { _tag: "ValidationFailed"; errors: string[] }

// Handle errors at appropriate boundaries
const workflow = pipe(
  fetchData(),
  Effect.catchTag("NotFound", () => useDefault()),
  Effect.catchTag("ValidationFailed", errors => logAndFail(errors))
)
```

### 2. Service Composition

```typescript
// Keep services focused
interface UserService {
  readonly getUser: (id: string) => Effect.Effect<User, UserError>
}

interface AuthService {
  readonly checkPermission: (permission: string) => Effect.Effect<boolean>
}

// Compose in application logic
const authorizedFetch = Effect.gen(function* () {
  const auth = yield* AuthService
  const users = yield* UserService
  
  const canView = yield* auth.checkPermission("view:users")
  if (!canView) return yield* Effect.fail({ _tag: "Unauthorized" })
  
  return yield* users.getUser(id)
})
```

### 3. Testing Strategy

```typescript
// Create test layers
const TestLayer = Layer.mergeAll(
  MockUserService,
  MockAuthService
)

// Test with mocks
const test = pipe(
  applicationLogic,
  Effect.provide(TestLayer)
)

await Effect.runPromise(test)
```

### 4. Performance

```typescript
// Use caching for expensive operations
const userCache = Cache.make({
  capacity: 100,
  timeToLive: Duration.minutes(5),
  lookup: fetchUser
})

// Batch requests
const users = Effect.all(
  ids.map(getUser),
  { batching: true }
)
```

---

## CI/CD Integration

### GitHub Actions Example

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: "npm"
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Type check
        run: npm run type-check
      
      - name: Test
        run: npm run test
      
      - name: Build
        run: npm run build
```

### Package Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "type-check": "tsc --noEmit",
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx}\""
  }
}
```

---

## Environment Variables

### React/Vite

Create `.env`:

```bash
VITE_API_BASE_URL=http://localhost:3000/api
VITE_ENABLE_LOGGING=true
```

Access in code:

```typescript
const config = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL,
  enableLogging: import.meta.env.VITE_ENABLE_LOGGING === "true"
}
```

### SvelteKit

Create `.env`:

```bash
PUBLIC_API_BASE_URL=http://localhost:3000/api
PRIVATE_API_KEY=secret
```

Access in code:

```typescript
import { PUBLIC_API_BASE_URL } from "$env/static/public"
import { PRIVATE_API_KEY } from "$env/static/private"
```

---

## Deployment

### Vercel

```bash
npm install -g vercel
vercel
```

### Netlify

```bash
npm install -g netlify-cli
netlify deploy
```

### Docker

Create `Dockerfile`:

```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --production

EXPOSE 3000
CMD ["npm", "run", "preview"]
```

---

This setup guide provides everything you need to start a production-ready Effect-TS project with React or Svelte!
