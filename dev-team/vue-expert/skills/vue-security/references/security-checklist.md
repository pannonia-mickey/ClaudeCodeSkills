# Security Checklist

## OWASP Vue Checklist

### Template Security
- [ ] Never use `v-html` with user-supplied content
- [ ] If `v-html` is required, sanitize with DOMPurify
- [ ] Never construct templates from user input with `Vue.compile()`
- [ ] Avoid dynamic component names from user input
- [ ] Escape user content in `href`, `src`, and event handlers

```vue
<!-- DANGEROUS: Dynamic href can execute javascript: -->
<a :href="userUrl">Link</a>

<!-- SAFE: Validate URL protocol -->
<script setup lang="ts">
const safeUrl = computed(() => {
  try {
    const url = new URL(props.userUrl)
    return ['http:', 'https:', 'mailto:'].includes(url.protocol) ? url.href : '#'
  } catch {
    return '#'
  }
})
</script>
<a :href="safeUrl">Link</a>
```

### State and Data
- [ ] Never store secrets in client-side state (Pinia, ref, localStorage)
- [ ] Clear sensitive data from state on logout
- [ ] Validate all data from URL params and query strings
- [ ] Sanitize data before rendering in templates

```ts
// Clear all stores on logout
function logout() {
  const pinia = getActivePinia()
  if (pinia) {
    Object.keys(pinia.state.value).forEach(storeId => {
      const store = pinia._s.get(storeId)
      store?.$reset()
    })
  }
  navigateTo('/login')
}
```

### API Communication
- [ ] Always use HTTPS in production
- [ ] Include CSRF tokens for state-changing requests
- [ ] Validate API responses match expected schema
- [ ] Handle 401/403 responses with token refresh or redirect
- [ ] Set appropriate timeouts on fetch requests

## Dependency Scanning

```bash
# npm audit for vulnerability scanning
npm audit
npm audit fix

# Automated scanning in CI
npm audit --audit-level=high

# Use Snyk for deeper analysis
npx snyk test
npx snyk monitor

# Lock file integrity
npm ci  # Uses exact versions from package-lock.json
```

```yaml
# .github/workflows/security.yml
name: Security
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm audit --audit-level=high
      - name: Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

## Environment Variables

```ts
// NEVER expose secrets to the client

// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // Server-only — SAFE for secrets
    dbPassword: process.env.DB_PASSWORD,
    jwtSecret: process.env.JWT_SECRET,
    apiKey: process.env.API_KEY,

    // Public — NEVER put secrets here
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE,
      appName: 'My App',
    },
  },
})

// Vite .env files
// .env.local (gitignored)
VITE_API_BASE=https://api.example.com  // Exposed to client!
DB_PASSWORD=secret123                   // NOT exposed (no VITE_ prefix)

// Validate env at build time
// env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_BASE: string
}
```

## Secure Headers

```ts
// server/middleware/security-headers.ts
export default defineEventHandler((event) => {
  setResponseHeaders(event, {
    // Prevent MIME sniffing
    'X-Content-Type-Options': 'nosniff',

    // Prevent clickjacking
    'X-Frame-Options': 'DENY',

    // Control referrer information
    'Referrer-Policy': 'strict-origin-when-cross-origin',

    // Disable browser features you don't use
    'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',

    // Force HTTPS
    'Strict-Transport-Security': 'max-age=63072000; includeSubDomains; preload',

    // CSP
    'Content-Security-Policy': [
      "default-src 'self'",
      "script-src 'self'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "connect-src 'self' https://api.example.com",
      "font-src 'self'",
      "object-src 'none'",
      "base-uri 'self'",
      "form-action 'self'",
      "frame-ancestors 'none'",
    ].join('; '),
  })
})
```

## File Upload Safety

```vue
<script setup lang="ts">
const MAX_FILE_SIZE = 5 * 1024 * 1024 // 5MB
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf']

async function handleUpload(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file) return

  // Client-side validation
  if (file.size > MAX_FILE_SIZE) {
    error.value = 'File too large (max 5MB)'
    return
  }

  if (!ALLOWED_TYPES.includes(file.type)) {
    error.value = 'File type not allowed'
    return
  }

  // Validate file name
  const safeName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_')

  const formData = new FormData()
  formData.append('file', file, safeName)

  await $fetch('/api/upload', {
    method: 'POST',
    body: formData,
  })
}
</script>

<template>
  <input
    type="file"
    :accept="ALLOWED_TYPES.join(',')"
    @change="handleUpload"
  />
</template>
```

```ts
// server/api/upload.post.ts — server-side validation
import { readMultipartFormData } from 'h3'

export default defineEventHandler(async (event) => {
  const files = await readMultipartFormData(event)
  if (!files?.length) {
    throw createError({ statusCode: 400, message: 'No file provided' })
  }

  const file = files[0]

  // Server-side size check
  if (file.data.length > 5 * 1024 * 1024) {
    throw createError({ statusCode: 413, message: 'File too large' })
  }

  // Verify MIME type by checking magic bytes, not Content-Type header
  const fileType = await import('file-type')
  const detected = await fileType.fileTypeFromBuffer(file.data)

  if (!detected || !['image/jpeg', 'image/png', 'image/webp', 'application/pdf'].includes(detected.mime)) {
    throw createError({ statusCode: 400, message: 'Invalid file type' })
  }

  // Generate safe filename
  const ext = detected.ext
  const filename = `${crypto.randomUUID()}.${ext}`

  // Store file (never use user-provided filename)
  await storeFile(filename, file.data)

  return { filename, size: file.data.length }
})
```

## Third-Party Script Safety

```ts
// Load third-party scripts safely with useHead
useHead({
  script: [
    {
      src: 'https://analytics.example.com/script.js',
      defer: true,
      // Use SRI for integrity verification
      integrity: 'sha384-abc123...',
      crossorigin: 'anonymous',
    },
  ],
})

// Or use useScript (Nuxt)
const { load } = useScript({
  src: 'https://maps.googleapis.com/maps/api/js',
  defer: true,
}, {
  trigger: 'manual', // Don't load until needed
})

// Load only when component mounts
onMounted(() => load())
```
