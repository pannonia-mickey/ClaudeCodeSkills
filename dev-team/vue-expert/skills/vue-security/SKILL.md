---
name: Vue Security
description: This skill should be used when the user asks about "Vue security", "Vue XSS", "Vue sanitization", "Vue authentication", "Vue CSRF", "Vue CSP", "Nuxt auth", "Vue secure coding", or "Vue input validation". It covers XSS prevention, authentication patterns, CSP, input sanitization, and secure coding practices.
---

# Vue Security

## XSS Prevention

```vue
<script setup lang="ts">
// Vue auto-escapes template interpolation — SAFE
const userInput = ref('<script>alert("xss")</script>')
</script>

<template>
  <!-- SAFE: Auto-escaped by Vue -->
  <p>{{ userInput }}</p>
  <!-- Renders as text: <script>alert("xss")</script> -->

  <!-- DANGEROUS: v-html bypasses escaping -->
  <div v-html="userInput"></div>
  <!-- NEVER use v-html with user input -->

  <!-- SAFE: Use DOMPurify for trusted rich content -->
  <div v-html="sanitizedContent"></div>
</template>

<script setup lang="ts">
import DOMPurify from 'dompurify'

const rawContent = ref('<p>Hello <b>world</b></p><script>alert("xss")</script>')
const sanitizedContent = computed(() => DOMPurify.sanitize(rawContent.value))
</script>
```

## Input Validation

```vue
<script setup lang="ts">
import { z } from 'zod'

const userSchema = z.object({
  name: z.string().min(2).max(100).regex(/^[a-zA-Z\s'-]+$/),
  email: z.string().email().max(254),
  age: z.number().int().min(0).max(150).optional(),
  bio: z.string().max(1000).optional(),
  website: z.string().url().optional().or(z.literal('')),
})

type UserForm = z.infer<typeof userSchema>

const form = reactive<UserForm>({
  name: '',
  email: '',
  age: undefined,
  bio: '',
  website: '',
})

const errors = ref<Record<string, string>>({})

function validate(): boolean {
  const result = userSchema.safeParse(form)
  if (!result.success) {
    errors.value = Object.fromEntries(
      result.error.issues.map(issue => [issue.path[0], issue.message])
    )
    return false
  }
  errors.value = {}
  return true
}

async function handleSubmit() {
  if (!validate()) return
  await api.createUser(form)
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <div>
      <label for="name">Name</label>
      <input id="name" v-model="form.name" maxlength="100" />
      <span v-if="errors.name" class="error">{{ errors.name }}</span>
    </div>
    <div>
      <label for="email">Email</label>
      <input id="email" v-model="form.email" type="email" maxlength="254" />
      <span v-if="errors.email" class="error">{{ errors.email }}</span>
    </div>
    <button type="submit">Submit</button>
  </form>
</template>
```

## Authentication Pattern

```ts
// composables/useAuth.ts
export function useAuth() {
  const token = useCookie('auth-token', {
    httpOnly: false, // Must be false for client access; prefer httpOnly server-side cookie
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 24, // 1 day
  })

  const user = useState<User | null>('auth-user', () => null)
  const isAuthenticated = computed(() => !!token.value)

  async function login(email: string, password: string) {
    const response = await $fetch('/api/auth/login', {
      method: 'POST',
      body: { email, password },
    })
    token.value = response.token
    user.value = response.user
  }

  async function logout() {
    await $fetch('/api/auth/logout', { method: 'POST' })
    token.value = null
    user.value = null
    navigateTo('/login')
  }

  async function fetchUser() {
    if (!token.value) return
    try {
      user.value = await $fetch('/api/auth/me')
    } catch {
      token.value = null
      user.value = null
    }
  }

  return { token, user, isAuthenticated, login, logout, fetchUser }
}
```

## CSRF Protection

```ts
// server/middleware/csrf.ts (Nuxt)
export default defineEventHandler(async (event) => {
  if (['GET', 'HEAD', 'OPTIONS'].includes(event.method)) return

  const origin = getHeader(event, 'origin')
  const host = getHeader(event, 'host')

  // Verify origin matches host
  if (origin && new URL(origin).host !== host) {
    throw createError({ statusCode: 403, message: 'CSRF validation failed' })
  }
})

// Client-side: include custom header (cannot be set by forms)
const api = $fetch.create({
  onRequest({ options }) {
    options.headers = {
      ...options.headers,
      'X-Requested-With': 'XMLHttpRequest',
    }
  },
})
```

## Content Security Policy

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/**': {
      headers: {
        'Content-Security-Policy': [
          "default-src 'self'",
          "script-src 'self' 'nonce-{NONCE}'",
          "style-src 'self' 'unsafe-inline'",
          "img-src 'self' data: https:",
          "font-src 'self'",
          "connect-src 'self' https://api.example.com",
          "frame-ancestors 'none'",
        ].join('; '),
        'X-Content-Type-Options': 'nosniff',
        'X-Frame-Options': 'DENY',
        'Referrer-Policy': 'strict-origin-when-cross-origin',
      },
    },
  },
})
```

## References

- [Auth Patterns](references/auth-patterns.md) — OAuth flow, JWT refresh tokens, role-based access, protected routes, session management.
- [Security Checklist](references/security-checklist.md) — OWASP Vue checklist, dependency scanning, environment variables, secure headers, file upload safety.
