---
name: Vue State Management
description: This skill should be used when the user asks about "Nuxt 3", "Nuxt server", "Nuxt middleware", "Nuxt composables", "Nuxt data fetching", "useFetch Nuxt", "Nuxt SEO", "Nuxt deployment", or "Nuxt SSR". It covers Nuxt 3 framework patterns including server routes, middleware, data fetching, SEO, and deployment.
---

# Vue State Management — Nuxt 3

## Nuxt 3 Project Structure

```
nuxt-app/
├── app.vue                 # Root component
├── nuxt.config.ts          # Nuxt configuration
├── pages/                  # File-based routing
│   ├── index.vue           # /
│   ├── about.vue           # /about
│   └── users/
│       ├── index.vue       # /users
│       └── [id].vue        # /users/:id
├── components/             # Auto-imported components
│   ├── AppHeader.vue
│   └── ui/
│       └── Button.vue      # <UiButton />
├── composables/            # Auto-imported composables
│   └── useAuth.ts
├── server/                 # Server-side (Nitro)
│   ├── api/
│   │   └── users.get.ts    # GET /api/users
│   ├── middleware/
│   │   └── auth.ts
│   └── utils/
│       └── db.ts
├── middleware/              # Route middleware
│   └── auth.ts
├── layouts/
│   ├── default.vue
│   └── admin.vue
├── plugins/
│   └── api.ts
└── public/
    └── favicon.ico
```

## Nuxt Data Fetching

```vue
<!-- pages/users/[id].vue -->
<script setup lang="ts">
const route = useRoute()

// useFetch — SSR-friendly data fetching
const { data: user, pending, error, refresh } = await useFetch<User>(
  `/api/users/${route.params.id}`,
  {
    key: `user-${route.params.id}`,
    transform: (data) => ({
      ...data,
      fullName: `${data.firstName} ${data.lastName}`,
    }),
  }
)

// useAsyncData — for non-URL-based async operations
const { data: stats } = await useAsyncData(
  `user-stats-${route.params.id}`,
  () => $fetch(`/api/users/${route.params.id}/stats`)
)

// Lazy fetching — doesn't block navigation
const { data: recommendations, pending: loadingRecs } = useLazyFetch(
  `/api/users/${route.params.id}/recommendations`
)
</script>

<template>
  <div>
    <div v-if="pending">Loading...</div>
    <div v-else-if="error">Error: {{ error.message }}</div>
    <div v-else-if="user">
      <h1>{{ user.fullName }}</h1>
      <UserStats :stats="stats" />
      <div v-if="loadingRecs">Loading recommendations...</div>
      <RecommendationList v-else :items="recommendations" />
    </div>
  </div>
</template>
```

## Server Routes (Nitro)

```ts
// server/api/users.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const page = Number(query.page) || 1
  const limit = Number(query.limit) || 20

  const db = useDatabase()
  const users = await db.select().from(usersTable).limit(limit).offset((page - 1) * limit)

  return { items: users, page, limit }
})

// server/api/users.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody<CreateUserInput>(event)

  // Validate input
  if (!body.name || !body.email) {
    throw createError({ statusCode: 400, message: 'Name and email required' })
  }

  const db = useDatabase()
  const user = await db.insert(usersTable).values(body).returning()

  return user[0]
})

// server/api/users/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  const user = await db.select().from(usersTable).where(eq(usersTable.id, id)).limit(1)

  if (!user[0]) {
    throw createError({ statusCode: 404, message: 'User not found' })
  }

  return user[0]
})
```

## Nuxt Middleware

```ts
// middleware/auth.ts — named middleware
export default defineNuxtRouteMiddleware((to, from) => {
  const auth = useAuth()

  if (!auth.isAuthenticated.value) {
    return navigateTo('/login', { replace: true })
  }
})

// Apply in page
definePageMeta({
  middleware: 'auth',
  layout: 'admin',
})

// middleware/auth.global.ts — global middleware (runs on every route)
export default defineNuxtRouteMiddleware((to) => {
  const publicPages = ['/login', '/register', '/']
  if (publicPages.includes(to.path)) return

  const auth = useAuth()
  if (!auth.isAuthenticated.value) {
    return navigateTo('/login')
  }
})
```

## Nuxt SEO

```vue
<script setup lang="ts">
// Page-level SEO
useHead({
  title: 'User Profile',
  meta: [
    { name: 'description', content: 'View user profile and activity' },
    { property: 'og:title', content: 'User Profile' },
    { property: 'og:image', content: '/og-image.png' },
  ],
  link: [
    { rel: 'canonical', href: 'https://example.com/users/123' },
  ],
})

// Dynamic SEO from data
useSeoMeta({
  title: () => user.value?.name ?? 'Loading...',
  ogTitle: () => user.value?.name,
  description: () => `Profile for ${user.value?.name}`,
  ogImage: () => user.value?.avatarUrl,
})
</script>
```

## References

- [Nuxt Patterns](references/nuxt-patterns.md) — Plugins, server middleware, runtime config, caching, deployment strategies, error handling.
- [Nuxt Advanced](references/nuxt-advanced.md) — Custom server utils, WebSocket support, edge rendering, hybrid rendering, module development.
