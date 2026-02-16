# Nuxt Patterns

## Plugins

```ts
// plugins/api.ts — client + server plugin
export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()

  const api = $fetch.create({
    baseURL: config.public.apiBase,
    onRequest({ options }) {
      const auth = useAuth()
      if (auth.token.value) {
        options.headers = {
          ...options.headers,
          Authorization: `Bearer ${auth.token.value}`,
        }
      }
    },
    onResponseError({ response }) {
      if (response.status === 401) {
        navigateTo('/login')
      }
    },
  })

  return {
    provide: { api },
  }
})

// Usage in components
const { $api } = useNuxtApp()
const users = await $api('/api/users')

// plugins/error-handler.client.ts — client-only plugin
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.config.errorHandler = (error, instance, info) => {
    console.error('Vue error:', error)
    // Send to error tracking service
  }
})
```

## Runtime Config

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // Server-only (not exposed to client)
    databaseUrl: process.env.DATABASE_URL,
    jwtSecret: process.env.JWT_SECRET,

    // Public (available on client and server)
    public: {
      apiBase: process.env.API_BASE || 'http://localhost:3000',
      appName: 'My App',
    },
  },
})

// Usage in server routes
const config = useRuntimeConfig()
const db = createConnection(config.databaseUrl) // Server-only

// Usage in components
const config = useRuntimeConfig()
console.log(config.public.apiBase) // Available on client
// config.databaseUrl — undefined on client
```

## Server Middleware

```ts
// server/middleware/cors.ts
export default defineEventHandler((event) => {
  setResponseHeaders(event, {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
  })

  if (event.method === 'OPTIONS') {
    event.node.res.statusCode = 204
    event.node.res.end()
  }
})

// server/middleware/auth.ts
export default defineEventHandler(async (event) => {
  // Skip auth for non-API routes
  if (!event.path.startsWith('/api/') || event.path === '/api/auth/login') {
    return
  }

  const token = getHeader(event, 'authorization')?.replace('Bearer ', '')
  if (!token) {
    throw createError({ statusCode: 401, message: 'Unauthorized' })
  }

  try {
    const user = await verifyToken(token)
    event.context.user = user
  } catch {
    throw createError({ statusCode: 401, message: 'Invalid token' })
  }
})

// Access in route handlers
export default defineEventHandler((event) => {
  const user = event.context.user // Set by auth middleware
  // ...
})
```

## Caching

```ts
// Server-side caching with Nitro
// server/api/products.get.ts
export default defineCachedEventHandler(async () => {
  const products = await db.select().from(productsTable)
  return products
}, {
  maxAge: 60 * 10, // Cache for 10 minutes
  swr: true,        // Stale-while-revalidate
  name: 'products-list',
})

// Cache with varying keys
export default defineCachedEventHandler(async (event) => {
  const category = getQuery(event).category
  return await fetchProductsByCategory(category)
}, {
  maxAge: 300,
  varies: ['category'], // Different cache per category
})

// Client-side cache with useFetch
const { data } = await useFetch('/api/products', {
  getCachedData(key, nuxtApp) {
    return nuxtApp.payload.data[key] || nuxtApp.static.data[key]
  },
})
```

## Error Handling

```vue
<!-- error.vue — global error page -->
<script setup lang="ts">
const props = defineProps<{
  error: {
    statusCode: number
    message: string
    stack?: string
  }
}>()

const handleError = () => clearError({ redirect: '/' })
</script>

<template>
  <div class="error-page">
    <h1>{{ error.statusCode }}</h1>
    <p>{{ error.message }}</p>
    <button @click="handleError">Go Home</button>
  </div>
</template>
```

```vue
<!-- Component-level error boundary -->
<template>
  <NuxtErrorBoundary>
    <DataTable :items="items" />
    <template #error="{ error, clearError }">
      <div class="error-state">
        <p>Failed to render: {{ error.message }}</p>
        <button @click="clearError">Retry</button>
      </div>
    </template>
  </NuxtErrorBoundary>
</template>
```

## Deployment

```ts
// nuxt.config.ts — deployment presets
export default defineNuxtConfig({
  // Node.js server (default)
  nitro: { preset: 'node-server' },

  // Vercel
  // nitro: { preset: 'vercel' },

  // Cloudflare Pages
  // nitro: { preset: 'cloudflare-pages' },

  // Static generation
  // nitro: { preset: 'static' },
  // Or: nuxi generate

  // Docker
  // nitro: { preset: 'node-server' },
})
```

```dockerfile
# Dockerfile
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-slim
WORKDIR /app
COPY --from=build /app/.output ./.output
ENV NITRO_PORT=3000
EXPOSE 3000
CMD ["node", ".output/server/index.mjs"]
```
