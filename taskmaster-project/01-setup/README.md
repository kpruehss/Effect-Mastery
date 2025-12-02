# Chapter 1: Project Setup & Configuration

**Setting up the TaskMaster development environment**

In this chapter, we'll initialize our SvelteKit project, install Effect-TS and all necessary dependencies, and configure our development environment.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initialize SvelteKit](#initialize-sveltekit)
3. [Install Dependencies](#install-dependencies)
4. [Project Structure](#project-structure)
5. [Configuration Files](#configuration-files)
6. [Verify Setup](#verify-setup)
7. [Next Steps](#next-steps)

---

## Prerequisites

Before starting, ensure you have:

- **Node.js** 18+ installed
- **npm** or **pnpm** package manager
- A code editor (VS Code recommended)
- Basic terminal/command line knowledge

---

## Initialize SvelteKit

Let's create a new SvelteKit project:

```bash
# Create new project
npm create svelte@latest taskmaster

# During setup, choose:
# - Skeleton project
# - Yes, using TypeScript syntax
# - Add ESLint for code linting
# - Add Prettier for code formatting
# - Add Vitest for unit testing

# Navigate into project
cd taskmaster
```

### Initial Installation

```bash
# Install base dependencies
npm install
```

---

## Install Dependencies

### Core Dependencies

Install Effect-TS and related packages:

```bash
# Effect-TS core
npm install effect

# Effect platform (for HTTP, etc.)
npm install @effect/platform

# Effect Schema (for validation)
npm install @effect/schema
```

### Styling Dependencies

Install Tailwind CSS for styling:

```bash
# Install Tailwind
npm install -D tailwindcss postcss autoprefixer

# Initialize Tailwind config
npx tailwindcss init -p
```

### Testing Dependencies

Install additional testing tools:

```bash
npm install -D @testing-library/svelte @testing-library/jest-dom
```

### Complete Package.json

Your `package.json` should include:

```json
{
  "name": "taskmaster",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest",
    "check": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json",
    "check:watch": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json --watch",
    "lint": "eslint .",
    "format": "prettier --write ."
  },
  "devDependencies": {
    "@sveltejs/adapter-auto": "^3.0.0",
    "@sveltejs/kit": "^2.0.0",
    "@sveltejs/vite-plugin-svelte": "^4.0.0",
    "@testing-library/jest-dom": "^6.1.5",
    "@testing-library/svelte": "^4.0.5",
    "@types/eslint": "^8.56.0",
    "autoprefixer": "^10.4.16",
    "eslint": "^8.56.0",
    "postcss": "^8.4.32",
    "prettier": "^3.1.1",
    "svelte": "^5.0.0",
    "svelte-check": "^3.6.0",
    "tailwindcss": "^3.4.0",
    "tslib": "^2.4.1",
    "typescript": "^5.0.0",
    "vite": "^5.0.0",
    "vitest": "^1.0.0"
  },
  "dependencies": {
    "@effect/platform": "^0.63.0",
    "@effect/schema": "^0.72.0",
    "effect": "^3.7.0"
  },
  "type": "module"
}
```

---

## Project Structure

Create the recommended folder structure:

```bash
# Create directories
mkdir -p src/lib/{domain,services,effects,stores,components}

# Create initial files
touch src/lib/domain/.gitkeep
touch src/lib/services/.gitkeep
touch src/lib/effects/.gitkeep
touch src/lib/stores/.gitkeep
touch src/lib/components/.gitkeep
```

### Directory Layout

```
taskmaster/
├── src/
│   ├── lib/
│   │   ├── domain/          # Domain models and types
│   │   ├── services/        # Effect services
│   │   ├── effects/         # Effect layers and runtime
│   │   ├── stores/          # Svelte stores and runes helpers
│   │   └── components/      # Svelte components
│   ├── routes/              # SvelteKit routes
│   │   └── +page.svelte     # Home page
│   └── app.html             # HTML template
├── static/                  # Static assets
├── tests/                   # Test files
├── package.json
├── svelte.config.js
├── tsconfig.json
├── vite.config.ts
└── tailwind.config.js
```

---

## Configuration Files

### 1. SvelteKit Configuration

Update `svelte.config.js`:

```javascript
import adapter from '@sveltejs/adapter-auto'
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte'

/** @type {import('@sveltejs/kit').Config} */
const config = {
  preprocess: vitePreprocess(),

  kit: {
    adapter: adapter(),

    // Path aliases for cleaner imports
    alias: {
      '$lib': './src/lib',
      '$domain': './src/lib/domain',
      '$services': './src/lib/services',
      '$effects': './src/lib/effects',
      '$stores': './src/lib/stores',
      '$components': './src/lib/components'
    }
  }
}

export default config
```

### 2. TypeScript Configuration

Update `tsconfig.json`:

```json
{
  "extends": "./.svelte-kit/tsconfig.json",
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "sourceMap": true,
    "strict": true,
    "moduleResolution": "bundler",
    "paths": {
      "$lib": ["./src/lib"],
      "$lib/*": ["./src/lib/*"],
      "$domain": ["./src/lib/domain"],
      "$domain/*": ["./src/lib/domain/*"],
      "$services": ["./src/lib/services"],
      "$services/*": ["./src/lib/services/*"],
      "$effects": ["./src/lib/effects"],
      "$effects/*": ["./src/lib/effects/*"],
      "$stores": ["./src/lib/stores"],
      "$stores/*": ["./src/lib/stores/*"],
      "$components": ["./src/lib/components"],
      "$components/*": ["./src/lib/components/*"]
    }
  }
}
```

### 3. Tailwind CSS Configuration

Update `tailwind.config.js`:

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./src/**/*.{html,js,svelte,ts}'],
  theme: {
    extend: {
      colors: {
        // Custom color palette
        primary: {
          50: '#f0f9ff',
          100: '#e0f2fe',
          500: '#0ea5e9',
          600: '#0284c7',
          700: '#0369a1',
        }
      }
    }
  },
  plugins: []
}
```

### 4. Create Tailwind CSS Entry

Create `src/app.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom base styles */
@layer base {
  body {
    @apply bg-gray-50 text-gray-900;
  }
}

/* Custom component styles */
@layer components {
  .btn {
    @apply px-4 py-2 rounded-lg font-medium transition-colors;
  }

  .btn-primary {
    @apply bg-primary-600 text-white hover:bg-primary-700;
  }

  .card {
    @apply bg-white border border-gray-200 rounded-lg shadow-sm;
  }
}
```

### 5. Update Root Layout

Update `src/routes/+layout.svelte`:

```svelte
<script lang="ts">
  import '../app.css'
</script>

<slot />
```

### 6. Vite Configuration

Update `vite.config.ts`:

```typescript
import { sveltekit } from '@sveltejs/kit/vite'
import { defineConfig } from 'vitest/config'

export default defineConfig({
  plugins: [sveltekit()],

  test: {
    include: ['src/**/*.{test,spec}.{js,ts}'],
    globals: true,
    environment: 'jsdom'
  },

  optimizeDeps: {
    include: ['effect']
  }
})
```

### 7. Environment Variables

Create `.env.example`:

```bash
# API Configuration
VITE_API_URL=http://localhost:3000/api
VITE_WS_URL=ws://localhost:3000/ws

# Feature Flags
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_DARK_MODE=true
```

Create `.env` (gitignored):

```bash
# Copy from .env.example and customize
VITE_API_URL=http://localhost:3000/api
VITE_WS_URL=ws://localhost:3000/ws
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_DARK_MODE=true
```

### 8. Update .gitignore

Add to `.gitignore`:

```
.env
.env.local
.vercel
```

---

## Verify Setup

### 1. Create Test Page

Update `src/routes/+page.svelte`:

```svelte
<script lang="ts">
  import { Effect } from 'effect'

  // Simple Effect test
  const testEffect = Effect.succeed('Effect-TS is working! 🎉')
  const result = Effect.runSync(testEffect)
</script>

<div class="min-h-screen flex items-center justify-center">
  <div class="card p-8 max-w-md">
    <h1 class="text-3xl font-bold text-primary-600 mb-4">
      TaskMaster
    </h1>
    <p class="text-gray-600 mb-4">
      SvelteKit + Effect-TS Project Setup Complete
    </p>
    <div class="bg-green-50 border border-green-200 rounded-lg p-4">
      <p class="text-green-800 font-medium">
        {result}
      </p>
    </div>
  </div>
</div>
```

### 2. Run Development Server

```bash
npm run dev
```

Visit `http://localhost:5173` - you should see the test page with the success message.

### 3. Run Type Checking

```bash
npm run check
```

No errors should appear.

### 4. Run Linting

```bash
npm run lint
```

Should complete without errors.

---

## Troubleshooting

### Common Issues

**Issue: Effect import errors**
```bash
# Solution: Ensure Effect is installed
npm install effect
```

**Issue: Path alias not resolving**
```bash
# Solution: Restart VS Code and run
npm run check
```

**Issue: Tailwind not working**
```bash
# Solution: Ensure app.css is imported in +layout.svelte
# and Tailwind config content path is correct
```

**Issue: TypeScript errors**
```bash
# Solution: Sync SvelteKit types
npx svelte-kit sync
```

---

## Project Checklist

Before moving to Chapter 2, verify:

- ✅ SvelteKit project initialized
- ✅ Effect-TS dependencies installed
- ✅ Tailwind CSS configured and working
- ✅ Path aliases configured
- ✅ Folder structure created
- ✅ Test page displays correctly
- ✅ No TypeScript errors
- ✅ No linting errors

---

## What We Accomplished

In this chapter, we:

1. ✅ Created a new SvelteKit project with TypeScript
2. ✅ Installed Effect-TS and all dependencies
3. ✅ Configured Tailwind CSS for styling
4. ✅ Set up path aliases for clean imports
5. ✅ Created the project folder structure
6. ✅ Configured development tools (ESLint, Prettier, Vitest)
7. ✅ Verified the setup with a test page

---

## Next Steps

Now that our development environment is ready, we'll move to [Chapter 2: Domain Models & Types](../02-domain/README.md) where we'll:

- Define the Task domain model
- Create the Project model
- Set up User types
- Define domain error types
- Implement validation with Effect Schema

[Continue to Chapter 2 →](../02-domain/README.md)

---

[← Back to Main](../README.md) | [Chapter 2 →](../02-domain/README.md)
