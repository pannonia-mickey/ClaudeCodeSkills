# Caching Deep Dive

## Next.js Caching Layers

```
┌──────────────────────────────────────────────────────────┐
│ Layer              │ What                │ Duration       │
├──────────────────────────────────────────────────────────┤
│ Request Memoization│ Dedupe same fetch   │ Single request │
│                    │ in component tree   │ (server render)│
├──────────────────────────────────────────────────────────┤
│ Data Cache         │ fetch() results     │ Persistent     │
│                    │ stored on server    │ (until revalid)│
├──────────────────────────────────────────────────────────┤
│ Full Route Cache   │ Pre-rendered HTML   │ Persistent     │
│                    │ + RSC payload       │ (until revalid)│
├──────────────────────────────────────────────────────────┤
│ Router Cache       │ RSC payload in      │ 30s (dynamic)  │
│ (Client)           │ browser memory      │ 5min (static)  │
└──────────────────────────────────────────────────────────┘
```

## Request Memoization

```tsx
// Same fetch URL in multiple components → one request
// This happens automatically within a single server render

async function getUser(id: string) {
  // This fetch is automatically deduplicated
  const res = await fetch(`https://api.example.com/users/${id}`);
  return res.json();
}

// Component A calls getUser('123')
// Component B calls getUser('123')
// → Only ONE network request is made

// Important: Only works with fetch()
// For database queries, use React.cache():
import { cache } from 'react';

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});
// Now getUser('123') called in multiple components → one DB query
```

## Data Cache Controls

```tsx
// Default: cached indefinitely
fetch('https://api.example.com/data');

// Revalidate every 60 seconds (ISR)
fetch('https://api.example.com/data', {
  next: { revalidate: 60 },
});

// No caching (SSR — always fresh)
fetch('https://api.example.com/data', {
  cache: 'no-store',
});

// Tag-based revalidation
fetch('https://api.example.com/products', {
  next: { tags: ['products'] },
});
// Later: revalidateTag('products')

// Page-level cache control
export const revalidate = 60;      // Revalidate every 60s
export const dynamic = 'force-dynamic'; // No caching
export const dynamic = 'force-static';  // Force static
export const fetchCache = 'default-no-store'; // All fetches dynamic
```

## Full Route Cache

```
Static Routes (pre-rendered at build time):
  ✓ No dynamic functions (cookies(), headers(), searchParams)
  ✓ All fetch calls cached
  ✓ HTML + RSC Payload stored

Dynamic Routes (rendered per request):
  ✗ Uses cookies(), headers(), or searchParams
  ✗ Any fetch with cache: 'no-store'
  ✗ export const dynamic = 'force-dynamic'

// Making a static route dynamic:
// Any of these makes the whole route dynamic:
import { cookies, headers } from 'next/headers';

export default async function Page() {
  const cookieStore = await cookies();    // → dynamic
  const headersList = await headers();    // → dynamic
  // Or: searchParams in page props        → dynamic
}

// Making a dynamic route static:
export const dynamic = 'force-static';
// Useful for pages that use cookies() but you know the data is safe to cache
```

## Router Cache (Client-Side)

```tsx
// The router cache stores previously visited routes in the browser

// Prefetching behavior:
// <Link> — prefetches automatically when visible in viewport
// router.prefetch('/path') — programmatic prefetch
// <Link prefetch={false}> — disable prefetch

// Cache duration:
// Static pages: 5 minutes
// Dynamic pages: 30 seconds

// Invalidate router cache:
import { useRouter } from 'next/navigation';

const router = useRouter();
router.refresh(); // Invalidate current route's router cache
// router.push() after a Server Action automatically invalidates

// Full cache invalidation from Server Action:
'use server';
import { revalidatePath } from 'next/cache';

export async function updateData() {
  await db.update(/* ... */);
  revalidatePath('/dashboard'); // Invalidates Data Cache + Router Cache
}
```

## Caching with Database Queries (no fetch)

```tsx
// unstable_cache for non-fetch data sources
import { unstable_cache } from 'next/cache';

const getCachedUser = unstable_cache(
  async (id: string) => {
    return db.user.findUnique({ where: { id } });
  },
  ['user'],  // Cache key prefix
  {
    tags: ['users'],
    revalidate: 3600, // 1 hour
  }
);

// Usage in Server Component
export default async function UserProfile({ userId }: { userId: string }) {
  const user = await getCachedUser(userId);
  return <div>{user.name}</div>;
}

// Invalidate:
revalidateTag('users');
```

## Cache Debugging

```javascript
// next.config.js — log cache behavior
module.exports = {
  logging: {
    fetches: {
      fullUrl: true,  // Log full fetch URLs
      // Shows: cache: HIT, cache: MISS, cache: SKIP
    },
  },
};

// In browser DevTools:
// Network tab → filter by RSC → inspect __next_data requests
// Look for x-nextjs-cache header: HIT, MISS, STALE
```
