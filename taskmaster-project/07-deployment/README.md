# Chapter 7: Deployment & Production

**Preparing TaskMaster for production deployment**

In this chapter, we'll prepare our application for production, configure deployment settings, and deploy to various platforms.

---

## Table of Contents

1. [Production Checklist](#production-checklist)
2. [Environment Configuration](#environment-configuration)
3. [Build Optimization](#build-optimization)
4. [Deployment Platforms](#deployment-platforms)
5. [Monitoring & Error Tracking](#monitoring--error-tracking)
6. [Performance Optimization](#performance-optimization)
7. [Security Hardening](#security-hardening)
8. [Next Steps](#next-steps)

---

## Production Checklist

Before deploying to production, ensure:

- ✅ All tests pass (`npm run test`)
- ✅ Type checking succeeds (`npm run check`)
- ✅ No linting errors (`npm run lint`)
- ✅ Build completes successfully (`npm run build`)
- ✅ Environment variables are configured
- ✅ API endpoints are production-ready
- ✅ Error handling is comprehensive
- ✅ Security headers are configured
- ✅ Performance is optimized

---

## Environment Configuration

### 1. Environment Variables

Create production environment configuration:

**`.env.production`:**

```bash
# API Configuration
VITE_API_URL=https://api.taskmaster.app/api
VITE_WS_URL=wss://api.taskmaster.app/ws

# Feature Flags
VITE_ENABLE_ANALYTICS=true
VITE_ENABLE_DARK_MODE=true

# Deployment
PUBLIC_APP_VERSION=1.0.0
PUBLIC_BUILD_TIME=${BUILD_TIME}
```

### 2. SvelteKit Configuration

Update `svelte.config.js` for production:

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
      '$effects': './src/lib/effects',
      '$stores': './src/lib/stores',
      '$components': './src/lib/components'
    },

    // Production CSP headers
    csp: {
      mode: 'auto',
      directives: {
        'script-src': ['self'],
        'style-src': ['self', 'unsafe-inline'],
        'img-src': ['self', 'data:', 'https:'],
        'font-src': ['self'],
        'connect-src': ['self', process.env.VITE_API_URL, process.env.VITE_WS_URL]
      }
    }
  }
}

export default config
```

### 3. Runtime Configuration

Create `src/lib/config.ts`:

```typescript
import { Effect, Layer, Context } from 'effect'

/**
 * Application configuration
 */
export interface AppConfig {
  readonly apiUrl: string
  readonly wsUrl: string
  readonly enableAnalytics: boolean
  readonly enableDarkMode: boolean
  readonly appVersion: string
  readonly environment: 'development' | 'production' | 'test'
}

export class AppConfig extends Context.Tag('AppConfig')<
  AppConfig,
  AppConfig
>() {}

/**
 * Load configuration from environment variables
 */
const loadConfig = Effect.sync((): AppConfig => {
  const apiUrl = import.meta.env.VITE_API_URL
  const wsUrl = import.meta.env.VITE_WS_URL

  if (!apiUrl) {
    throw new Error('VITE_API_URL is required')
  }

  if (!wsUrl) {
    throw new Error('VITE_WS_URL is required')
  }

  return {
    apiUrl,
    wsUrl,
    enableAnalytics: import.meta.env.VITE_ENABLE_ANALYTICS === 'true',
    enableDarkMode: import.meta.env.VITE_ENABLE_DARK_MODE === 'true',
    appVersion: import.meta.env.PUBLIC_APP_VERSION || '0.0.0',
    environment: import.meta.env.MODE as 'development' | 'production' | 'test'
  }
})

/**
 * Configuration Layer
 */
export const AppConfigLive = Layer.effect(
  AppConfig,
  loadConfig
)
```

---

## Build Optimization

### 1. Vite Configuration

Update `vite.config.ts` for production:

```typescript
import { sveltekit } from '@sveltejs/kit/vite'
import { defineConfig } from 'vitest/config'

export default defineConfig(({ mode }) => ({
  plugins: [sveltekit()],

  test: {
    include: ['src/**/*.{test,spec}.{js,ts}'],
    globals: true,
    environment: 'jsdom'
  },

  build: {
    // Optimize for production
    minify: 'esbuild',
    sourcemap: mode === 'production' ? false : true,

    rollupOptions: {
      output: {
        manualChunks: {
          // Separate Effect bundle
          'effect': ['effect', '@effect/platform', '@effect/schema']
        }
      }
    },

    // Increase chunk size warning limit
    chunkSizeWarningLimit: 1000
  },

  optimizeDeps: {
    include: ['effect']
  }
}))
```

### 2. Code Splitting

Implement route-based code splitting in `src/routes/+layout.ts`:

```typescript
export const prerender = false
export const ssr = true

// Preload critical data
export const load = async () => {
  return {
    // Return minimal data needed for layout
  }
}
```

### 3. Asset Optimization

Create `static/.htaccess` for caching:

```apache
<IfModule mod_expires.c>
  ExpiresActive On

  # Images
  ExpiresByType image/jpeg "access plus 1 year"
  ExpiresByType image/png "access plus 1 year"
  ExpiresByType image/svg+xml "access plus 1 year"
  ExpiresByType image/webp "access plus 1 year"

  # CSS and JavaScript
  ExpiresByType text/css "access plus 1 month"
  ExpiresByType application/javascript "access plus 1 month"

  # Fonts
  ExpiresByType font/woff2 "access plus 1 year"
</IfModule>
```

---

## Deployment Platforms

### Option 1: Vercel (Recommended)

**1. Install Vercel Adapter:**

```bash
npm install -D @sveltejs/adapter-vercel
```

**2. Update `svelte.config.js`:**

```javascript
import adapter from '@sveltejs/adapter-vercel'

const config = {
  kit: {
    adapter: adapter({
      runtime: 'nodejs20.x',
      regions: ['iad1'], // Choose your region
      split: false
    })
  }
}
```

**3. Deploy:**

```bash
# Install Vercel CLI
npm install -g vercel

# Deploy
vercel

# Production deployment
vercel --prod
```

**4. Environment Variables:**

Set in Vercel dashboard:
- `VITE_API_URL`
- `VITE_WS_URL`
- `VITE_ENABLE_ANALYTICS`

---

### Option 2: Netlify

**1. Install Netlify Adapter:**

```bash
npm install -D @sveltejs/adapter-netlify
```

**2. Update `svelte.config.js`:**

```javascript
import adapter from '@sveltejs/adapter-netlify'

const config = {
  kit: {
    adapter: adapter({
      edge: false,
      split: false
    })
  }
}
```

**3. Create `netlify.toml`:**

```toml
[build]
  command = "npm run build"
  publish = "build"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[build.environment]
  NODE_VERSION = "20"
```

**4. Deploy:**

```bash
# Install Netlify CLI
npm install -g netlify-cli

# Deploy
netlify deploy

# Production deployment
netlify deploy --prod
```

---

### Option 3: Docker

**1. Create `Dockerfile`:**

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:20-alpine

WORKDIR /app

# Copy built application
COPY --from=builder /app/build ./build
COPY --from=builder /app/package*.json ./

# Install production dependencies only
RUN npm ci --omit=dev

# Expose port
EXPOSE 3000

# Start application
CMD ["node", "build"]
```

**2. Create `docker-compose.yml`:**

```yaml
version: '3.8'

services:
  taskmaster:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - VITE_API_URL=${VITE_API_URL}
      - VITE_WS_URL=${VITE_WS_URL}
    restart: unless-stopped
```

**3. Build and Run:**

```bash
# Build image
docker build -t taskmaster .

# Run container
docker run -p 3000:3000 taskmaster

# Or use docker-compose
docker-compose up -d
```

---

### Option 4: Node Server

**1. Install Node Adapter:**

```bash
npm install -D @sveltejs/adapter-node
```

**2. Update `svelte.config.js`:**

```javascript
import adapter from '@sveltejs/adapter-node'

const config = {
  kit: {
    adapter: adapter({
      out: 'build',
      precompress: true,
      envPrefix: 'VITE_'
    })
  }
}
```

**3. Build and Run:**

```bash
# Build
npm run build

# Run with Node
node build

# Run with PM2 (recommended)
npm install -g pm2
pm2 start build/index.js --name taskmaster
```

---

## Monitoring & Error Tracking

### 1. Sentry Integration

**Install Sentry:**

```bash
npm install @sentry/sveltekit
```

**Configure in `src/hooks.client.ts`:**

```typescript
import * as Sentry from '@sentry/sveltekit'

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  tracesSampleRate: 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0
})

export const handleError = Sentry.handleErrorWithSentry()
```

### 2. Analytics

**Create `src/lib/analytics.ts`:**

```typescript
import { Effect } from 'effect'
import { AppConfig } from '$lib/config'

export const trackPageView = (path: string) =>
  Effect.gen(function* (_) {
    const config = yield* _(AppConfig)

    if (!config.enableAnalytics) {
      return
    }

    // Send to analytics service
    if (typeof window !== 'undefined' && window.gtag) {
      window.gtag('event', 'page_view', {
        page_path: path
      })
    }
  })

export const trackEvent = (
  category: string,
  action: string,
  label?: string
) =>
  Effect.gen(function* (_) {
    const config = yield* _(AppConfig)

    if (!config.enableAnalytics) {
      return
    }

    if (typeof window !== 'undefined' && window.gtag) {
      window.gtag('event', action, {
        event_category: category,
        event_label: label
      })
    }
  })
```

### 3. Health Check Endpoint

**Create `src/routes/api/health/+server.ts`:**

```typescript
import { json } from '@sveltejs/kit'
import type { RequestHandler } from './$types'

export const GET: RequestHandler = async () => {
  return json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: import.meta.env.PUBLIC_APP_VERSION || '0.0.0'
  })
}
```

---

## Performance Optimization

### 1. Image Optimization

Use `@sveltejs/enhanced-img`:

```bash
npm install -D @sveltejs/enhanced-img
```

**Update `svelte.config.js`:**

```javascript
import { enhancedImages } from '@sveltejs/enhanced-img'

const config = {
  preprocess: vitePreprocess(),

  plugins: [enhancedImages()]
}
```

### 2. Lazy Loading Components

```svelte
<script lang="ts">
  import { onMount } from 'svelte'

  let HeavyComponent: any

  onMount(async () => {
    const module = await import('./HeavyComponent.svelte')
    HeavyComponent = module.default
  })
</script>

{#if HeavyComponent}
  <svelte:component this={HeavyComponent} />
{/if}
```

### 3. Prefetching

Enable prefetching in `src/routes/+layout.svelte`:

```svelte
<script lang="ts">
  import { beforeNavigate } from '$app/navigation'

  beforeNavigate(({ to }) => {
    // Prefetch data for next page
    if (to?.route.id) {
      // Trigger prefetch
    }
  })
</script>
```

---

## Security Hardening

### 1. Security Headers

Create `src/hooks.server.ts`:

```typescript
import type { Handle } from '@sveltejs/kit'

export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event)

  // Security headers
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-Content-Type-Options', 'nosniff')
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
  response.headers.set(
    'Permissions-Policy',
    'geolocation=(), microphone=(), camera=()'
  )
  response.headers.set(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains'
  )

  return response
}
```

### 2. CSRF Protection

```typescript
import { Effect } from 'effect'
import { randomBytes } from 'crypto'

export const generateCSRFToken = Effect.sync(() => {
  return randomBytes(32).toString('hex')
})

export const validateCSRFToken = (
  token: string,
  expectedToken: string
) =>
  Effect.sync(() => {
    if (token !== expectedToken) {
      throw new Error('Invalid CSRF token')
    }
  })
```

### 3. Rate Limiting

```typescript
import { Effect, Duration } from 'effect'
import type { RequestEvent } from '@sveltejs/kit'

const requestCounts = new Map<string, { count: number; resetAt: number }>()

export const rateLimit = (event: RequestEvent, maxRequests = 100) =>
  Effect.gen(function* (_) {
    const ip = event.getClientAddress()
    const now = Date.now()

    const current = requestCounts.get(ip)

    if (!current || current.resetAt < now) {
      requestCounts.set(ip, {
        count: 1,
        resetAt: now + Duration.toMillis(Duration.minutes(1))
      })
      return
    }

    if (current.count >= maxRequests) {
      throw new Error('Rate limit exceeded')
    }

    current.count++
  })
```

---

## Production Deployment Script

Create `scripts/deploy.sh`:

```bash
#!/bin/bash

set -e

echo "🚀 Starting deployment process..."

# Run tests
echo "📝 Running tests..."
npm run test

# Type checking
echo "🔍 Type checking..."
npm run check

# Linting
echo "✨ Linting..."
npm run lint

# Build
echo "🏗️  Building application..."
npm run build

# Deploy based on environment
if [ "$1" == "vercel" ]; then
  echo "📦 Deploying to Vercel..."
  vercel --prod
elif [ "$1" == "netlify" ]; then
  echo "📦 Deploying to Netlify..."
  netlify deploy --prod
elif [ "$1" == "docker" ]; then
  echo "🐳 Building Docker image..."
  docker build -t taskmaster:latest .
  docker push taskmaster:latest
else
  echo "❌ Unknown deployment target: $1"
  echo "Usage: ./scripts/deploy.sh [vercel|netlify|docker]"
  exit 1
fi

echo "✅ Deployment complete!"
```

Make it executable:

```bash
chmod +x scripts/deploy.sh
```

---

## Testing Production Build Locally

```bash
# Build for production
npm run build

# Preview production build
npm run preview

# Test with production environment variables
VITE_API_URL=https://api.taskmaster.app/api \
VITE_WS_URL=wss://api.taskmaster.app/ws \
npm run preview
```

---

## Post-Deployment Checklist

After deploying to production:

- ✅ Verify application loads correctly
- ✅ Test all critical user flows
- ✅ Check error tracking is working (Sentry)
- ✅ Verify analytics are tracking (if enabled)
- ✅ Test WebSocket connections
- ✅ Confirm API endpoints are accessible
- ✅ Check performance metrics
- ✅ Verify security headers
- ✅ Test on multiple devices/browsers
- ✅ Monitor server logs for errors

---

## Rollback Procedure

If something goes wrong:

**Vercel:**
```bash
# List deployments
vercel ls

# Rollback to previous deployment
vercel rollback [deployment-url]
```

**Netlify:**
```bash
# Rollback via CLI
netlify rollback

# Or use the Netlify dashboard
```

**Docker:**
```bash
# Revert to previous image
docker pull taskmaster:previous
docker-compose up -d
```

---

## Monitoring Commands

```bash
# Check application health
curl https://taskmaster.app/api/health

# Monitor logs (Vercel)
vercel logs

# Monitor logs (PM2)
pm2 logs taskmaster

# Monitor logs (Docker)
docker logs -f taskmaster
```

---

## What We Accomplished

In this chapter, we:

1. ✅ Configured environment variables for production
2. ✅ Optimized build configuration
3. ✅ Set up deployment for multiple platforms
4. ✅ Integrated monitoring and error tracking
5. ✅ Implemented performance optimizations
6. ✅ Added security hardening measures
7. ✅ Created deployment scripts
8. ✅ Established rollback procedures

---

## Congratulations! 🎉

You've successfully built and deployed a production-ready task management application with:

- **SvelteKit** for the frontend framework
- **Svelte 5 runes** for reactive state
- **Effect-TS** for functional programming patterns
- **Type-safe** domain models and services
- **Real-time** collaboration features
- **Production-ready** deployment configuration

---

## Next Steps

### Further Learning

1. **Effect-TS Deep Dive**: Explore advanced Effect patterns
2. **Performance Tuning**: Optimize bundle size and load times
3. **Feature Expansion**: Add more features to TaskMaster
4. **Testing**: Expand test coverage
5. **Accessibility**: Improve a11y compliance

### Additional Resources

- [Effect Documentation](https://effect.website/docs)
- [SvelteKit Documentation](https://kit.svelte.dev/docs)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
- [Web.dev Performance](https://web.dev/performance/)

---

[← Back to Chapter 6](../06-routes/README.md) | [Back to Main](../README.md)

**Thank you for completing the TaskMaster project! 🚀**
