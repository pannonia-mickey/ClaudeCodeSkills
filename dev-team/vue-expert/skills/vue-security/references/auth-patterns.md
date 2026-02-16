# Auth Patterns

## OAuth Flow (Nuxt 3)

```ts
// server/api/auth/github.get.ts — initiate OAuth
export default defineEventHandler((event) => {
  const config = useRuntimeConfig()
  const params = new URLSearchParams({
    client_id: config.github.clientId,
    redirect_uri: `${config.public.appUrl}/api/auth/github/callback`,
    scope: 'user:email',
    state: generateCSRFToken(),
  })
  return sendRedirect(event, `https://github.com/login/oauth/authorize?${params}`)
})

// server/api/auth/github/callback.get.ts — handle callback
export default defineEventHandler(async (event) => {
  const query = getQuery(event)

  // Exchange code for token
  const tokenResponse = await $fetch('https://github.com/login/oauth/access_token', {
    method: 'POST',
    body: {
      client_id: config.github.clientId,
      client_secret: config.github.clientSecret,
      code: query.code,
    },
    headers: { Accept: 'application/json' },
  })

  // Fetch user profile
  const githubUser = await $fetch('https://api.github.com/user', {
    headers: { Authorization: `Bearer ${tokenResponse.access_token}` },
  })

  // Create or update local user
  const user = await upsertUser({
    provider: 'github',
    providerId: String(githubUser.id),
    name: githubUser.name,
    email: githubUser.email,
    avatarUrl: githubUser.avatar_url,
  })

  // Create session
  const session = await createSession(user.id)
  setCookie(event, 'session', session.token, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7, // 7 days
  })

  return sendRedirect(event, '/dashboard')
})
```

## JWT Refresh Token Pattern

```ts
// server/api/auth/login.post.ts
export default defineEventHandler(async (event) => {
  const { email, password } = await readBody(event)
  const user = await authenticateUser(email, password)

  const accessToken = signJWT({ userId: user.id, role: user.role }, '15m')
  const refreshToken = signJWT({ userId: user.id }, '7d')

  // Store refresh token in httpOnly cookie
  setCookie(event, 'refresh_token', refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    path: '/api/auth/refresh',
    maxAge: 60 * 60 * 24 * 7,
  })

  return { accessToken, user }
})

// server/api/auth/refresh.post.ts
export default defineEventHandler(async (event) => {
  const refreshToken = getCookie(event, 'refresh_token')
  if (!refreshToken) {
    throw createError({ statusCode: 401, message: 'No refresh token' })
  }

  const payload = verifyJWT(refreshToken)
  const user = await getUserById(payload.userId)

  const newAccessToken = signJWT({ userId: user.id, role: user.role }, '15m')
  return { accessToken: newAccessToken }
})

// composables/useAuth.ts — auto-refresh on client
export function useAuth() {
  const accessToken = useState<string | null>('access-token', () => null)

  const api = $fetch.create({
    async onRequest({ options }) {
      if (accessToken.value) {
        options.headers = {
          ...options.headers,
          Authorization: `Bearer ${accessToken.value}`,
        }
      }
    },
    async onResponseError({ response }) {
      if (response.status === 401) {
        // Try refresh
        try {
          const { accessToken: newToken } = await $fetch('/api/auth/refresh', {
            method: 'POST',
          })
          accessToken.value = newToken
        } catch {
          accessToken.value = null
          navigateTo('/login')
        }
      }
    },
  })

  return { accessToken, api }
}
```

## Role-Based Access Control

```ts
// composables/useAuthorization.ts
type Role = 'admin' | 'editor' | 'viewer'

const permissions: Record<Role, string[]> = {
  admin: ['users:read', 'users:write', 'users:delete', 'settings:write'],
  editor: ['users:read', 'users:write'],
  viewer: ['users:read'],
}

export function useAuthorization() {
  const auth = useAuth()

  function hasPermission(permission: string): boolean {
    const role = auth.user.value?.role as Role
    return role ? permissions[role]?.includes(permission) ?? false : false
  }

  function hasRole(role: Role): boolean {
    return auth.user.value?.role === role
  }

  return { hasPermission, hasRole }
}
```

```vue
<!-- v-if based on permissions -->
<script setup lang="ts">
const { hasPermission } = useAuthorization()
</script>

<template>
  <button v-if="hasPermission('users:delete')" @click="deleteUser(user.id)">
    Delete
  </button>
</template>
```

```ts
// Directive for permission checks
// plugins/auth-directive.ts
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.directive('permission', {
    mounted(el, binding) {
      const { hasPermission } = useAuthorization()
      if (!hasPermission(binding.value)) {
        el.remove()
      }
    },
  })
})

// Usage: <button v-permission="'users:delete'">Delete</button>
```

## Protected Routes

```ts
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to) => {
  const auth = useAuth()

  if (!auth.isAuthenticated.value) {
    return navigateTo({
      path: '/login',
      query: { redirect: to.fullPath },
    })
  }
})

// middleware/role.ts
export default defineNuxtRouteMiddleware((to) => {
  const auth = useAuth()
  const requiredRole = to.meta.role as string

  if (requiredRole && auth.user.value?.role !== requiredRole) {
    return navigateTo('/forbidden')
  }
})

// In page:
definePageMeta({
  middleware: ['auth', 'role'],
  role: 'admin',
})
```

## Session Management

```ts
// server/utils/session.ts
import { randomBytes } from 'crypto'

interface Session {
  token: string
  userId: string
  createdAt: Date
  expiresAt: Date
}

export async function createSession(userId: string): Promise<Session> {
  const token = randomBytes(32).toString('hex')
  const session: Session = {
    token,
    userId,
    createdAt: new Date(),
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  }

  // Store in database or Redis
  await db.insert(sessionsTable).values(session)
  return session
}

export async function validateSession(token: string): Promise<Session | null> {
  const session = await db.select().from(sessionsTable)
    .where(eq(sessionsTable.token, token))
    .limit(1)

  if (!session[0] || new Date(session[0].expiresAt) < new Date()) {
    return null
  }

  return session[0]
}

export async function invalidateSession(token: string): Promise<void> {
  await db.delete(sessionsTable).where(eq(sessionsTable.token, token))
}

// Invalidate all sessions for a user (password change, security event)
export async function invalidateAllSessions(userId: string): Promise<void> {
  await db.delete(sessionsTable).where(eq(sessionsTable.userId, userId))
}
```
