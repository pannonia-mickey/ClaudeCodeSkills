# Middleware Patterns

## Authentication Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { getToken } from 'next-auth/jwt';

const publicPaths = ['/login', '/register', '/forgot-password', '/api/auth'];
const adminPaths = ['/admin'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Skip public paths
  if (publicPaths.some((p) => pathname.startsWith(p))) {
    return NextResponse.next();
  }

  // Verify JWT token
  const token = await getToken({
    req: request,
    secret: process.env.NEXTAUTH_SECRET,
  });

  if (!token) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', pathname);
    return NextResponse.redirect(loginUrl);
  }

  // Role-based access
  if (adminPaths.some((p) => pathname.startsWith(p))) {
    if (token.role !== 'admin') {
      return NextResponse.redirect(new URL('/unauthorized', request.url));
    }
  }

  // Add user info to headers for downstream use
  const response = NextResponse.next();
  response.headers.set('x-user-id', token.sub as string);
  response.headers.set('x-user-role', token.role as string);
  return response;
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|public).*)'],
};
```

## Rate Limiting Middleware

```typescript
// Edge-compatible rate limiting using headers
import { NextResponse } from 'next/server';

const rateLimit = new Map<string, { count: number; resetAt: number }>();

export function middleware(request: NextRequest) {
  if (!request.nextUrl.pathname.startsWith('/api')) {
    return NextResponse.next();
  }

  const ip = request.headers.get('x-forwarded-for') || 'unknown';
  const now = Date.now();
  const windowMs = 60_000; // 1 minute
  const maxRequests = 60;

  const current = rateLimit.get(ip);
  if (!current || now > current.resetAt) {
    rateLimit.set(ip, { count: 1, resetAt: now + windowMs });
    return NextResponse.next();
  }

  current.count++;
  if (current.count > maxRequests) {
    return NextResponse.json(
      { error: 'Too many requests' },
      {
        status: 429,
        headers: {
          'Retry-After': String(Math.ceil((current.resetAt - now) / 1000)),
          'X-RateLimit-Limit': String(maxRequests),
          'X-RateLimit-Remaining': '0',
        },
      }
    );
  }

  const response = NextResponse.next();
  response.headers.set('X-RateLimit-Remaining', String(maxRequests - current.count));
  return response;
}

// Note: For production, use Upstash Redis or Vercel KV
// for distributed rate limiting across instances
```

## Geo-Routing Middleware

```typescript
export function middleware(request: NextRequest) {
  const country = request.geo?.country || 'US';
  const { pathname } = request.nextUrl;

  // Country-specific content
  if (pathname === '/pricing') {
    const currency = getCurrency(country);
    const response = NextResponse.next();
    response.headers.set('x-user-country', country);
    response.headers.set('x-user-currency', currency);
    return response;
  }

  // EU GDPR consent check
  const euCountries = ['DE', 'FR', 'IT', 'ES', 'NL', 'BE', 'AT', 'PL'];
  if (euCountries.includes(country)) {
    const hasConsent = request.cookies.get('cookie-consent')?.value;
    if (!hasConsent && !pathname.startsWith('/cookie-policy')) {
      // Show cookie banner
      const response = NextResponse.next();
      response.headers.set('x-show-cookie-banner', 'true');
      return response;
    }
  }

  return NextResponse.next();
}
```

## Feature Flag Middleware

```typescript
export function middleware(request: NextRequest) {
  const bucket = request.cookies.get('feature-bucket')?.value;

  if (!bucket) {
    // Assign user to feature bucket
    const response = NextResponse.next();
    const features = {
      'new-checkout': Math.random() < 0.1,  // 10% rollout
      'dark-mode': Math.random() < 0.5,     // 50% rollout
    };
    response.cookies.set('feature-bucket', JSON.stringify(features), {
      httpOnly: true,
      maxAge: 60 * 60 * 24 * 7, // 1 week
    });
    return response;
  }

  // Rewrite to feature variant
  const features = JSON.parse(bucket);
  if (features['new-checkout'] && request.nextUrl.pathname === '/checkout') {
    return NextResponse.rewrite(new URL('/checkout-v2', request.url));
  }

  return NextResponse.next();
}
```

## Bot Detection / SEO

```typescript
export function middleware(request: NextRequest) {
  const userAgent = request.headers.get('user-agent') || '';
  const isBot = /bot|crawl|spider|slurp|googlebot|bingbot/i.test(userAgent);

  if (isBot) {
    // Serve pre-rendered version for bots
    const response = NextResponse.next();
    response.headers.set('x-is-bot', 'true');
    // Could rewrite to a pre-rendered URL
    return response;
  }

  return NextResponse.next();
}
```

## Composing Multiple Middleware

```typescript
// Chain middleware functions
type MiddlewareFn = (
  request: NextRequest,
  response: NextResponse
) => NextResponse | Response | null;

function chain(middlewares: MiddlewareFn[]) {
  return (request: NextRequest) => {
    let response = NextResponse.next();

    for (const middleware of middlewares) {
      const result = middleware(request, response);
      if (result instanceof Response && result.status !== 200) {
        return result; // Short-circuit on redirect/error
      }
      if (result) response = result as NextResponse;
    }

    return response;
  };
}

export default chain([
  authMiddleware,
  rateLimitMiddleware,
  geoMiddleware,
  loggingMiddleware,
]);
```
