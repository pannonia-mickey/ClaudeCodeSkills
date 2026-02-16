---
name: App Router Patterns
description: This skill should be used when the user asks about "parallel routes", "intercepting routes", "route groups", "dynamic segments", "catch-all routes", "Next.js middleware", "route handlers", "Next.js API routes", or "App Router patterns". It covers advanced App Router routing patterns and middleware.
---

# App Router Patterns

## Parallel Routes

```tsx
// Parallel routes render multiple pages in the same layout simultaneously
// app/dashboard/layout.tsx
export default function Layout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div>
      {children}
      <div className="grid grid-cols-2 gap-4">
        {analytics}
        {team}
      </div>
    </div>
  );
}

// app/dashboard/@analytics/page.tsx
export default async function AnalyticsSlot() {
  const data = await getAnalytics();
  return <AnalyticsChart data={data} />;
}

// app/dashboard/@analytics/loading.tsx
export default function AnalyticsLoading() {
  return <Skeleton className="h-64" />;
}

// app/dashboard/@team/page.tsx
export default async function TeamSlot() {
  const members = await getTeamMembers();
  return <TeamList members={members} />;
}

// Each slot loads independently — fast slots appear first
```

## Intercepting Routes

```tsx
// Intercept navigation to show modal, keep context
// app/feed/page.tsx — photo feed
// app/feed/(..)photo/[id]/page.tsx — intercepts /photo/[id] when navigating from feed
// app/photo/[id]/page.tsx — full page for direct URL access

// app/feed/(..)photo/[id]/page.tsx
export default function PhotoModal({ params }: { params: { id: string } }) {
  return (
    <Modal>
      <PhotoDetail id={params.id} />
    </Modal>
  );
}

// Interception conventions:
// (.)   — same level
// (..)  — one level up
// (..)(..) — two levels up
// (...) — from root
```

## Route Handlers (API Routes)

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '10');

  const users = await db.user.findMany({
    skip: (page - 1) * limit,
    take: limit,
  });

  return NextResponse.json({ data: users, page, limit });
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const validated = createUserSchema.safeParse(body);

  if (!validated.success) {
    return NextResponse.json(
      { error: validated.error.flatten() },
      { status: 400 }
    );
  }

  const user = await db.user.create({ data: validated.data });
  return NextResponse.json(user, { status: 201 });
}

// app/api/users/[id]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const user = await db.user.findUnique({ where: { id } });

  if (!user) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }

  return NextResponse.json(user);
}
```

## Middleware

```typescript
// middleware.ts (root of project)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Authentication check
  const token = request.cookies.get('session-token')?.value;
  if (pathname.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // Locale detection
  const locale = request.headers.get('accept-language')?.split(',')[0]?.split('-')[0] || 'en';
  if (!pathname.startsWith('/api') && !pathname.match(/^\/(en|es|fr)/)) {
    return NextResponse.redirect(new URL(`/${locale}${pathname}`, request.url));
  }

  // A/B testing
  const bucket = request.cookies.get('ab-bucket')?.value;
  if (!bucket) {
    const response = NextResponse.next();
    response.cookies.set('ab-bucket', Math.random() > 0.5 ? 'a' : 'b');
    return response;
  }

  // Add headers
  const response = NextResponse.next();
  response.headers.set('x-request-id', crypto.randomUUID());
  return response;
}

export const config = {
  matcher: [
    // Match all except static files and images
    '/((?!_next/static|_next/image|favicon.ico|public).*)',
  ],
};
```

## Dynamic Routes

```tsx
// app/blog/[slug]/page.tsx — dynamic segment
export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}

// app/shop/[...categories]/page.tsx — catch-all
// Matches: /shop/clothes, /shop/clothes/tops, /shop/clothes/tops/t-shirts
export default async function Category({ params }: { params: Promise<{ categories: string[] }> }) {
  const { categories } = await params;
  // categories = ['clothes', 'tops', 't-shirts']
}

// app/docs/[[...slug]]/page.tsx — optional catch-all
// Matches: /docs, /docs/intro, /docs/advanced/setup
```

## References

- [Routing Patterns](references/routing-patterns.md) — Modal patterns, breadcrumbs, pagination, search params, navigation guards, back button handling.
- [Middleware Patterns](references/middleware-patterns.md) — Auth middleware, rate limiting, geo-routing, feature flags, bot detection, request logging.
