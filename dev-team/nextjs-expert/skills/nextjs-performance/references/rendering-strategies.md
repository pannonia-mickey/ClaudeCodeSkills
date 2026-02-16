# Rendering Strategies

## Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│ Strategy │ When to Use                   │ Trade-offs         │
├──────────────────────────────────────────────────────────────┤
│ SSG      │ Content rarely changes        │ Fast, but stale    │
│          │ Blog posts, docs, marketing   │ until rebuild      │
├──────────────────────────────────────────────────────────────┤
│ ISR      │ Content changes periodically  │ Fast + fresh,      │
│          │ Product pages, listings       │ stale window       │
├──────────────────────────────────────────────────────────────┤
│ SSR      │ Personalized/real-time data   │ Always fresh,      │
│          │ Dashboard, user profile       │ slower TTFB        │
├──────────────────────────────────────────────────────────────┤
│ CSR      │ Highly interactive, no SEO    │ Fast interactions, │
│          │ Admin panels, internal tools  │ no SSR benefit     │
├──────────────────────────────────────────────────────────────┤
│ PPR      │ Mixed static + dynamic        │ Best of both,      │
│          │ E-commerce with personalized  │ experimental       │
│          │ sections                      │                    │
├──────────────────────────────────────────────────────────────┤
│ Streaming│ Pages with varying data speed │ Progressive load,  │
│          │ Dashboards, search results    │ good UX            │
└──────────────────────────────────────────────────────────────┘
```

## SSG — Static Site Generation

```tsx
// Build-time rendering — fastest possible delivery
// app/blog/[slug]/page.tsx

export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

// Page is pre-rendered to HTML at build time
export default async function BlogPost({ params }: Props) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}

// Control behavior for unknown paths
export const dynamicParams = true;
// true:  Generate on first request, then cache (SSG on demand)
// false: Return 404 for paths not in generateStaticParams
```

## ISR — Incremental Static Regeneration

```tsx
// Time-based revalidation
export const revalidate = 60; // Regenerate every 60 seconds

// Or per-fetch revalidation
async function getProducts() {
  return fetch('https://api.example.com/products', {
    next: { revalidate: 3600 }, // 1 hour
  });
}

// On-demand revalidation (triggered by webhooks/actions)
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(request: NextRequest) {
  const { type, slug } = await request.json();

  if (type === 'product') {
    revalidateTag('products');
    revalidatePath(`/products/${slug}`);
  }

  return NextResponse.json({ revalidated: true });
}

// ISR Flow:
// 1. User requests /products/shoes
// 2. Serve cached version (instant)
// 3. If stale (> revalidate seconds), regenerate in background
// 4. Next request gets fresh version
// 5. Old version served until new one is ready (stale-while-revalidate)
```

## Streaming SSR

```tsx
// Stream page sections as they become ready
import { Suspense } from 'react';

export default function SearchResults({ searchParams }: Props) {
  const query = searchParams.q;

  return (
    <div>
      {/* Instant — static header */}
      <h1>Results for "{query}"</h1>

      {/* Streams at ~100ms */}
      <Suspense fallback={<FiltersSkeleton />}>
        <SearchFilters query={query} />
      </Suspense>

      {/* Streams at ~300ms */}
      <Suspense fallback={<ResultsSkeleton />}>
        <SearchResultsList query={query} />
      </Suspense>

      {/* Streams at ~500ms */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <RelatedProducts query={query} />
      </Suspense>
    </div>
  );
}

// User sees progressive loading:
// 0ms:   Static shell + skeletons
// 100ms: Filters appear
// 300ms: Results appear
// 500ms: Recommendations appear
```

## Edge Rendering

```tsx
// Run at the edge (closer to users, lower latency)
export const runtime = 'edge';

// Limitations of Edge Runtime:
// ✗ No Node.js APIs (fs, child_process, etc.)
// ✗ No native modules
// ✗ Limited to Web APIs
// ✓ Faster cold starts than Node.js
// ✓ Deployed to CDN edge locations
// ✓ Lower latency for global users

// Best for:
// - Simple data transformations
// - API proxying
// - A/B testing
// - Geo-based content
// - Auth token verification

// Use Node.js runtime for:
// - Database connections (Prisma, Drizzle)
// - File system operations
// - Heavy computation
// - Native modules
```

## Hybrid Approach

```tsx
// Mix strategies on a single page

export default function ProductPage({ params }: Props) {
  return (
    <div>
      {/* SSG — product info rarely changes */}
      <ProductDetails id={params.id} />

      {/* ISR — reviews update periodically */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews id={params.id} />
      </Suspense>

      {/* SSR — personalized pricing */}
      <Suspense fallback={<PriceSkeleton />}>
        <PersonalizedPrice id={params.id} />
      </Suspense>

      {/* CSR — interactive stock checker */}
      <StockChecker id={params.id} />
    </div>
  );
}

// PersonalizedPrice uses cookies() → makes route dynamic
// But Suspense allows static parts to be served instantly
// This is essentially what PPR formalizes
```
