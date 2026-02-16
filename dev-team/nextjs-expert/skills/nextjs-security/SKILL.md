---
name: Next.js Security
description: This skill should be used when the user asks about "Next.js security", "Server Actions validation", "Next.js CSRF", "Next.js CSP", "Next.js middleware auth", "Next.js environment variables", "Next.js rate limiting", "NextAuth.js", or "Next.js security headers". It covers Server Action security, authentication, CSP headers, and secure coding in Next.js.
---

# Next.js Security

## Server Action Security

```tsx
// ALWAYS validate and authorize in Server Actions
'use server';

import { z } from 'zod';
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';

const updateSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1).max(200),
  content: z.string().max(10000),
});

export async function updatePost(formData: FormData) {
  // 1. Authenticate
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  // 2. Validate input
  const parsed = updateSchema.safeParse({
    id: formData.get('id'),
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!parsed.success) {
    return { error: 'Invalid input', details: parsed.error.flatten() };
  }

  // 3. Authorize (check ownership)
  const post = await db.post.findUnique({ where: { id: parsed.data.id } });
  if (!post || post.authorId !== session.user.id) {
    throw new Error('Forbidden');
  }

  // 4. Perform action
  await db.post.update({
    where: { id: parsed.data.id },
    data: { title: parsed.data.title, content: parsed.data.content },
  });

  revalidatePath(`/posts/${parsed.data.id}`);
  return { success: true };
}

// Server Actions are POST endpoints — Next.js handles CSRF automatically
// But ALWAYS validate on the server, never trust client input
```

## Content Security Policy

```typescript
// middleware.ts — CSP with nonce
import { NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');

  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`,
    `style-src 'self' 'nonce-${nonce}'`,
    `img-src 'self' blob: data: https://images.example.com`,
    `font-src 'self'`,
    `connect-src 'self' https://api.example.com`,
    `frame-ancestors 'none'`,
    `form-action 'self'`,
    `base-uri 'self'`,
    `object-src 'none'`,
  ].join('; ');

  const response = NextResponse.next();
  response.headers.set('Content-Security-Policy', csp);
  response.headers.set('x-nonce', nonce);
  return response;
}

// app/layout.tsx — pass nonce to scripts
import { headers } from 'next/headers';
import Script from 'next/script';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const headersList = await headers();
  const nonce = headersList.get('x-nonce') || '';

  return (
    <html>
      <body>
        {children}
        <Script src="https://analytics.example.com" nonce={nonce} strategy="afterInteractive" />
      </body>
    </html>
  );
}
```

## Authentication with NextAuth.js v5

```typescript
// lib/auth.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import Google from 'next-auth/providers/google';
import { z } from 'zod';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    }),
    Credentials({
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      authorize: async (credentials) => {
        const parsed = z.object({
          email: z.string().email(),
          password: z.string().min(8),
        }).safeParse(credentials);

        if (!parsed.success) return null;

        const user = await db.user.findUnique({ where: { email: parsed.data.email } });
        if (!user) return null;

        const valid = await bcrypt.compare(parsed.data.password, user.passwordHash);
        if (!valid) return null;

        return { id: user.id, name: user.name, email: user.email, role: user.role };
      },
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) token.role = user.role;
      return token;
    },
    session({ session, token }) {
      session.user.id = token.sub!;
      session.user.role = token.role as string;
      return session;
    },
  },
  pages: {
    signIn: '/login',
    error: '/login',
  },
});

// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/lib/auth';
export const { GET, POST } = handlers;
```

## Security Headers

```javascript
// next.config.js
module.exports = {
  async headers() {
    return [{
      source: '/(.*)',
      headers: [
        { key: 'X-DNS-Prefetch-Control', value: 'on' },
        { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
        { key: 'X-Frame-Options', value: 'SAMEORIGIN' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=(), browsing-topics=()' },
        { key: 'X-XSS-Protection', value: '0' }, // Disabled — CSP handles this
      ],
    }];
  },
};
```

## Environment Variable Safety

```typescript
// NEVER expose server secrets to the client
// ❌ NEXT_PUBLIC_DATABASE_URL — exposed to browser!
// ❌ NEXT_PUBLIC_API_SECRET — exposed to browser!
// ✅ DATABASE_URL — server only
// ✅ NEXT_PUBLIC_APP_URL — safe, public info

// Validate at build time
import { z } from 'zod';

const serverSchema = z.object({
  DATABASE_URL: z.string().url(),
  NEXTAUTH_SECRET: z.string().min(32),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
});

const clientSchema = z.object({
  NEXT_PUBLIC_APP_URL: z.string().url(),
  NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: z.string().startsWith('pk_'),
});

// Throws at startup if misconfigured
export const serverEnv = serverSchema.parse(process.env);
export const clientEnv = clientSchema.parse({
  NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
  NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
});
```

## References

- [Auth Patterns](references/auth-patterns.md) — Session management, protected routes, role-based access, OAuth flows, JWT best practices.
- [Security Checklist](references/security-checklist.md) — OWASP Next.js checklist, common vulnerabilities, secure data handling, third-party script safety.
