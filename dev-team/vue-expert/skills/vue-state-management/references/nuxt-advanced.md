# Nuxt Advanced

## Custom Server Utils

```ts
// server/utils/db.ts — auto-imported in server routes
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'

let _db: ReturnType<typeof drizzle> | null = null

export function useDatabase() {
  if (!_db) {
    const config = useRuntimeConfig()
    const client = postgres(config.databaseUrl)
    _db = drizzle(client)
  }
  return _db
}

// server/utils/validate.ts
import { z, type ZodSchema } from 'zod'

export async function validateBody<T extends ZodSchema>(
  event: H3Event,
  schema: T
): Promise<z.infer<T>> {
  const body = await readBody(event)
  const result = schema.safeParse(body)

  if (!result.success) {
    throw createError({
      statusCode: 400,
      data: result.error.flatten(),
      message: 'Validation failed',
    })
  }

  return result.data
}

// Usage in route handler
const createUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
})

export default defineEventHandler(async (event) => {
  const body = await validateBody(event, createUserSchema)
  // body is typed as { name: string; email: string }
  return await createUser(body)
})
```

## WebSocket Support

```ts
// server/routes/_ws.ts — Nitro WebSocket
export default defineWebSocketHandler({
  open(peer) {
    console.log('Client connected:', peer.id)
    peer.subscribe('chat')
  },

  message(peer, message) {
    const data = JSON.parse(message.text())

    // Broadcast to all subscribers
    peer.publish('chat', JSON.stringify({
      user: peer.id,
      message: data.message,
      timestamp: Date.now(),
    }))
  },

  close(peer) {
    console.log('Client disconnected:', peer.id)
  },
})

// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    experimental: { websocket: true },
  },
})
```

```vue
<!-- Client-side WebSocket composable -->
<script setup lang="ts">
const { status, data, send, close } = useWebSocket('ws://localhost:3000/_ws')

const messages = ref<ChatMessage[]>([])

watch(data, (raw) => {
  if (raw) {
    messages.value.push(JSON.parse(raw))
  }
})

function sendMessage(text: string) {
  send(JSON.stringify({ message: text }))
}
</script>
```

## Hybrid Rendering

```ts
// nuxt.config.ts — per-route rendering rules
export default defineNuxtConfig({
  routeRules: {
    // Static pages — pre-rendered at build time
    '/': { prerender: true },
    '/about': { prerender: true },

    // ISR — regenerate every 60 seconds
    '/blog/**': { isr: 60 },

    // SWR — serve stale, revalidate in background
    '/products/**': { swr: 3600 },

    // SSR — always server-rendered (default)
    '/dashboard/**': { ssr: true },

    // Client-only — SPA mode
    '/admin/**': { ssr: false },

    // API caching
    '/api/products': { cache: { maxAge: 300 } },

    // CORS for API routes
    '/api/**': {
      cors: true,
      headers: { 'Access-Control-Allow-Origin': '*' },
    },

    // Redirect
    '/old-page': { redirect: '/new-page' },
  },
})
```

## Edge Rendering

```ts
// Deploy to edge with Cloudflare Workers
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
  },

  // Edge-compatible features
  routeRules: {
    '/': { prerender: true },
    '/api/**': { cors: true },
    '/dashboard/**': { ssr: true }, // Runs on edge
  },
})

// server/api/geo.get.ts — edge-optimized route
export default defineEventHandler((event) => {
  // Cloudflare provides geo data
  const cf = event.node.req.cf
  return {
    country: cf?.country,
    city: cf?.city,
    timezone: cf?.timezone,
  }
})
```

## Module Development

```ts
// modules/analytics/index.ts
import { defineNuxtModule, addPlugin, createResolver } from '@nuxt/kit'

export interface ModuleOptions {
  trackingId: string
  debug?: boolean
}

export default defineNuxtModule<ModuleOptions>({
  meta: {
    name: 'analytics',
    configKey: 'analytics',
  },
  defaults: {
    debug: false,
  },
  setup(options, nuxt) {
    const { resolve } = createResolver(import.meta.url)

    // Add runtime config
    nuxt.options.runtimeConfig.public.analytics = {
      trackingId: options.trackingId,
      debug: options.debug!,
    }

    // Add plugin
    addPlugin(resolve('./runtime/plugin'))

    // Add composable
    addImports({
      name: 'useAnalytics',
      from: resolve('./runtime/composables/useAnalytics'),
    })

    // Add server middleware
    addServerHandler({
      route: '/api/analytics',
      handler: resolve('./runtime/server/api/analytics.post'),
    })
  },
})

// Usage in nuxt.config.ts
export default defineNuxtConfig({
  modules: ['./modules/analytics'],
  analytics: {
    trackingId: 'UA-123456',
    debug: true,
  },
})
```

## SEO and Performance

```ts
// nuxt.config.ts — comprehensive SEO setup
export default defineNuxtConfig({
  app: {
    head: {
      htmlAttrs: { lang: 'en' },
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
      title: 'My App',
      meta: [
        { name: 'description', content: 'My awesome Nuxt app' },
        { name: 'theme-color', content: '#4f46e5' },
      ],
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' },
      ],
    },
  },

  // Performance
  experimental: {
    payloadExtraction: true,   // Extract payload for static pages
    renderJsonPayloads: true,  // Faster hydration
    componentIslands: true,    // Server-only components
  },
})
```
