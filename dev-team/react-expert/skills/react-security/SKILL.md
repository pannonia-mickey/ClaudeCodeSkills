---
name: React Security
description: This skill should be used when the user asks about "React XSS", "React security", "Content Security Policy", "dangerouslySetInnerHTML", "Next.js security", "auth token storage", "React CSRF", "secure React forms", or "frontend security". It covers XSS prevention, secure authentication patterns, CSP configuration, Next.js security hardening, and dependency auditing for React applications.
---

## XSS Prevention

### React's Built-in Escaping

React auto-escapes all string values rendered in JSX. This is safe by default:

```typescript
// Safe — React escapes the value
const userInput = '<script>alert("xss")</script>';
return <p>{userInput}</p>;
// Renders: &lt;script&gt;alert("xss")&lt;/script&gt;
```

### dangerouslySetInnerHTML

Never use `dangerouslySetInnerHTML` with unsanitized input. Always sanitize with DOMPurify:

```typescript
import DOMPurify from 'dompurify';

interface RichContentProps {
  html: string;
}

function RichContent({ html }: RichContentProps) {
  const sanitized = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'br', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li', 'h2', 'h3'],
    ALLOWED_ATTR: ['href', 'title', 'target'],
    ALLOW_DATA_ATTR: false,
  });

  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

### URL Sanitization

User-controlled `href` values can execute JavaScript:

```typescript
// DANGEROUS — allows javascript: protocol
<a href={userProvidedUrl}>Click</a>

// SAFE — validate URL protocol
function SafeLink({ href, children }: { href: string; children: React.ReactNode }) {
  const isValid = /^https?:\/\//i.test(href);
  if (!isValid) {
    return <span>{children}</span>;
  }
  return <a href={href} rel="noopener noreferrer">{children}</a>;
}
```

### Event Handler Injection

Never dynamically construct event handlers from user input:

```typescript
// DANGEROUS — user-controlled attribute
<div {...userControlledProps} />

// SAFE — allowlist specific props
function SafeDiv({ className, id }: { className?: string; id?: string }) {
  return <div className={className} id={id} />;
}
```

---

## Secure Authentication Token Storage

### Storage Comparison

| Storage | XSS Vulnerable | CSRF Vulnerable | Recommended |
|---------|---------------|-----------------|-------------|
| `localStorage` | Yes | No | No |
| `sessionStorage` | Yes | No | No |
| httpOnly cookie | No | Yes (mitigated with SameSite) | Yes |
| In-memory (variable) | No (lost on refresh) | No | For short sessions |

### httpOnly Cookie Pattern with Next.js

Set tokens as httpOnly cookies from the server:

```typescript
// app/api/auth/login/route.ts
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const { email, password } = await request.json();
  const { accessToken, refreshToken } = await authenticateUser(email, password);

  const cookieStore = await cookies();

  cookieStore.set('access_token', accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 15 * 60, // 15 minutes
    path: '/',
  });

  cookieStore.set('refresh_token', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 7 * 24 * 60 * 60, // 7 days
    path: '/api/auth',
  });

  return NextResponse.json({ success: true });
}
```

### Axios Interceptor for Token Refresh

```typescript
import axios from 'axios';

const api = axios.create({ baseURL: '/api', withCredentials: true });

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      await api.post('/auth/refresh');
      return api(originalRequest);
    }
    return Promise.reject(error);
  }
);
```

---

## Content Security Policy

### Next.js CSP via Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`,
    `style-src 'self' 'nonce-${nonce}'`,
    `img-src 'self' data: https:`,
    `font-src 'self'`,
    `connect-src 'self' https://api.example.com`,
    `frame-ancestors 'none'`,
    `base-uri 'self'`,
    `form-action 'self'`,
  ].join('; ');

  const response = NextResponse.next();
  response.headers.set('Content-Security-Policy', csp);
  response.headers.set('x-nonce', nonce);
  return response;
}
```

### Security Headers in next.config.js

```javascript
// next.config.js
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-XSS-Protection', value: '0' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains' },
];

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```

---

## Next.js Security Patterns

### Auth Middleware

Protect routes at the middleware level before they render:

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { verifyToken } from '@/lib/auth';

const protectedPaths = ['/dashboard', '/settings', '/api/protected'];

export async function middleware(request: NextRequest) {
  const isProtected = protectedPaths.some((p) => request.nextUrl.pathname.startsWith(p));
  if (!isProtected) return NextResponse.next();

  const token = request.cookies.get('access_token')?.value;
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  const payload = await verifyToken(token);
  if (!payload) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  const response = NextResponse.next();
  response.headers.set('x-user-id', payload.sub);
  return response;
}

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*', '/api/protected/:path*'],
};
```

### Environment Variable Exposure

Variables prefixed with `NEXT_PUBLIC_` are bundled into client JavaScript and visible to anyone:

```bash
# SAFE — server-only (never sent to browser)
DATABASE_URL=postgres://...
API_SECRET_KEY=sk-...

# EXPOSED — included in client bundle
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_STRIPE_PUBLIC_KEY=pk_live_...
```

Never put secrets, API keys, or database credentials in `NEXT_PUBLIC_` variables.

### Server Actions Security

Validate inputs in server actions — they are publicly callable HTTP endpoints:

```typescript
'use server';

import { z } from 'zod';
import { getSession } from '@/lib/auth';

const updateProfileSchema = z.object({
  name: z.string().min(1).max(100),
  bio: z.string().max(500).optional(),
});

export async function updateProfile(formData: FormData) {
  const session = await getSession();
  if (!session) throw new Error('Unauthorized');

  const parsed = updateProfileSchema.safeParse({
    name: formData.get('name'),
    bio: formData.get('bio'),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  await db.user.update({ where: { id: session.userId }, data: parsed.data });
  return { success: true };
}
```

---

## Dependency Security

### Audit and Monitor

```bash
# Check for known vulnerabilities
npm audit

# Fix automatically where possible
npm audit fix

# Check for outdated packages
npm outdated

# Use socket.dev for supply chain analysis
npx socket npm audit
```

### Lockfile Hygiene

- Always commit `package-lock.json` or `pnpm-lock.yaml`.
- Use `npm ci` (not `npm install`) in CI to install exact versions from the lockfile.
- Review lockfile diffs in pull requests for unexpected dependency changes.

---

## References

- **[XSS Prevention and CSP](references/xss-prevention.md)** — React escaping internals, DOMPurify configuration, SVG injection vectors, SSR XSS risks, CSP nonce and hash strategies, and CSP reporting.
- **[Authentication Patterns](references/auth-patterns.md)** — Token storage deep dive, httpOnly cookie implementation, Next.js auth middleware, OAuth2/OIDC with next-auth, CSRF protection, and session management.
