# Security Checklist

## Next.js Security Checklist

```markdown
## Server Components & Server Actions
- [ ] All Server Actions validate input with Zod/Valibot
- [ ] All Server Actions check authentication (auth())
- [ ] All Server Actions check authorization (user owns resource)
- [ ] Server-only modules use 'server-only' package import guard
- [ ] No sensitive data passed from Server to Client Components
- [ ] Server Actions don't expose internal error details to client

## Authentication
- [ ] NEXTAUTH_SECRET is strong (32+ random characters)
- [ ] OAuth redirect URIs are specific (not wildcards)
- [ ] Password requirements enforced (length, complexity)
- [ ] Account lockout after failed attempts
- [ ] Session tokens are httpOnly, secure, sameSite
- [ ] Token expiration is appropriate for the use case

## Environment Variables
- [ ] No secrets in NEXT_PUBLIC_ variables
- [ ] Environment variables validated at startup (Zod)
- [ ] .env.local is in .gitignore
- [ ] Production env vars set in deployment platform (not files)

## Headers & CSP
- [ ] Strict-Transport-Security header set
- [ ] X-Content-Type-Options: nosniff
- [ ] X-Frame-Options: DENY or SAMEORIGIN
- [ ] Content-Security-Policy configured with nonces
- [ ] Referrer-Policy set appropriately
- [ ] Permissions-Policy restricts unnecessary APIs

## API Routes / Route Handlers
- [ ] Authentication required for protected endpoints
- [ ] Rate limiting on auth endpoints
- [ ] Request body size limits configured
- [ ] CORS configured (not wildcard in production)
- [ ] Webhook endpoints verify signatures
- [ ] No SQL/NoSQL injection (parameterized queries)

## Data Handling
- [ ] User input sanitized before rendering (React handles this by default)
- [ ] dangerouslySetInnerHTML avoided (or input sanitized with DOMPurify)
- [ ] File uploads validate type, size, and content
- [ ] Sensitive data not logged (passwords, tokens, PII)
- [ ] Database queries use parameterized statements
- [ ] No sensitive data in URL parameters
```

## Common Vulnerabilities

### Server Component Data Leaks

```tsx
// ❌ BAD — passing sensitive data to Client Component
// app/page.tsx (Server Component)
const user = await db.user.findUnique({
  where: { id: session.user.id },
  // This includes passwordHash, which gets serialized to client!
});
return <ProfileCard user={user} />;

// ✅ GOOD — select only needed fields
const user = await db.user.findUnique({
  where: { id: session.user.id },
  select: { id: true, name: true, email: true, avatar: true },
});
return <ProfileCard user={user} />;
```

### Server-Only Module Guard

```typescript
// lib/db.ts — prevent accidental client import
import 'server-only'; // Throws build error if imported in Client Component

import { PrismaClient } from '@prisma/client';
export const db = new PrismaClient();

// This ensures db connection string never leaks to client bundle
```

### Unsafe HTML Rendering

```tsx
// ❌ BAD — XSS vulnerability
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// ✅ GOOD — sanitize first
import DOMPurify from 'isomorphic-dompurify';

<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />

// ✅ BETTER — use a markdown renderer with sanitization
import { remark } from 'remark';
import html from 'remark-html';
import sanitize from 'rehype-sanitize';

const processed = await remark().use(html).use(sanitize).process(content);
```

### Third-Party Script Safety

```tsx
// ❌ BAD — inline third-party script
<script src="https://unknown-cdn.com/widget.js" />

// ✅ GOOD — use next/script with strategy
import Script from 'next/script';

<Script
  src="https://known-analytics.com/script.js"
  strategy="lazyOnload"  // Load after page is interactive
  nonce={nonce}           // CSP nonce
  onError={() => {
    console.error('Analytics script failed to load');
  }}
/>

// Available strategies:
// beforeInteractive — load before hydration (rare)
// afterInteractive — load after hydration (default)
// lazyOnload — load during idle time (recommended for non-critical)
// worker — load in web worker (experimental)
```

### Secure File Uploads

```typescript
// actions/upload.ts
'use server';

const ALLOWED_TYPES = new Set(['image/jpeg', 'image/png', 'image/webp', 'application/pdf']);
const MAX_SIZE = 10 * 1024 * 1024; // 10MB

export async function uploadFile(formData: FormData) {
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  const file = formData.get('file') as File;

  // Validate file
  if (!file || file.size === 0) return { error: 'No file' };
  if (file.size > MAX_SIZE) return { error: 'File too large (max 10MB)' };
  if (!ALLOWED_TYPES.has(file.type)) return { error: 'Invalid file type' };

  // Validate content (don't trust Content-Type header alone)
  const bytes = new Uint8Array(await file.arrayBuffer());
  const magic = bytes.slice(0, 4);

  // Check magic bytes
  const isJpeg = magic[0] === 0xFF && magic[1] === 0xD8;
  const isPng = magic[0] === 0x89 && magic[1] === 0x50;
  const isPdf = magic[0] === 0x25 && magic[1] === 0x50;

  if (!isJpeg && !isPng && !isPdf) {
    return { error: 'File content does not match type' };
  }

  // Generate safe filename (never use user-provided filename)
  const ext = file.type.split('/')[1];
  const filename = `${crypto.randomUUID()}.${ext}`;

  // Upload to S3/storage (not local filesystem in production)
  await uploadToStorage(filename, Buffer.from(bytes));

  return { success: true, url: `/uploads/${filename}` };
}
```
