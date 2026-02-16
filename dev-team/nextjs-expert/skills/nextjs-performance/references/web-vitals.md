# Web Vitals Optimization

## Core Web Vitals Targets

```
┌──────────────────────────────────────────────────────────┐
│ Metric │ Good       │ Needs Work  │ Poor       │ Focus  │
├──────────────────────────────────────────────────────────┤
│ LCP    │ ≤ 2.5s     │ ≤ 4.0s      │ > 4.0s     │ Load   │
│ INP    │ ≤ 200ms    │ ≤ 500ms     │ > 500ms    │ Interact│
│ CLS    │ ≤ 0.1      │ ≤ 0.25      │ > 0.25     │ Stable │
└──────────────────────────────────────────────────────────┘

LCP = Largest Contentful Paint (loading speed)
INP = Interaction to Next Paint (responsiveness)
CLS = Cumulative Layout Shift (visual stability)
```

## LCP Optimization

```tsx
// 1. Preload LCP image
// next/image with priority handles this automatically
<Image src={heroImage} alt="Hero" priority />

// 2. Avoid render-blocking resources
// next/font handles font preloading with zero CLS
// next/script for deferred third-party scripts
import Script from 'next/script';
<Script src="https://analytics.example.com" strategy="lazyOnload" />

// 3. Server-render above-the-fold content
// Use Server Components (default in App Router)
// Avoid client-side data fetching for hero content

// 4. Optimize server response time
// Use edge runtime for latency-sensitive pages
export const runtime = 'edge';

// 5. Preconnect to required origins
// app/layout.tsx
export const metadata = {
  other: {
    'link': [
      { rel: 'preconnect', href: 'https://api.example.com' },
      { rel: 'dns-prefetch', href: 'https://cdn.example.com' },
    ],
  },
};
```

## INP Optimization

```tsx
// 1. Use transitions for non-urgent updates
'use client';
import { useTransition } from 'react';

function SearchResults({ query }: { query: string }) {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState([]);

  function handleSearch(value: string) {
    // Input update is urgent (immediate)
    setQuery(value);

    // Results update is non-urgent (can be deferred)
    startTransition(() => {
      setResults(filterProducts(value));
    });
  }

  return (
    <div>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending ? <Spinner /> : <ProductList items={results} />}
    </div>
  );
}

// 2. Defer expensive renders
import { useDeferredValue } from 'react';

function FilteredList({ filter }: { filter: string }) {
  const deferredFilter = useDeferredValue(filter);
  // Uses stale value during typing, updates when idle
  const items = useMemo(() => filterItems(deferredFilter), [deferredFilter]);
  return <ul>{items.map(/* ... */)}</ul>;
}

// 3. Virtualize long lists
import { useVirtualizer } from '@tanstack/react-virtual';
```

## CLS Optimization

```tsx
// 1. Always set dimensions on images
<Image src={photo} alt="Photo" width={800} height={600} />
// Or use fill with a sized container
<div className="relative aspect-video">
  <Image src={photo} alt="Photo" fill />
</div>

// 2. Use next/font (automatically prevents FOUT/FOIT)
const inter = Inter({ subsets: ['latin'], display: 'swap' });

// 3. Reserve space for dynamic content
// Loading skeleton with fixed dimensions
function CardSkeleton() {
  return <div className="h-48 w-full animate-pulse rounded bg-gray-200" />;
}

// 4. Avoid injecting content above existing content
// BAD: Banner that pushes content down after load
// GOOD: Fixed-height banner placeholder, or overlay

// 5. Use CSS containment
// .card { contain: layout; } — prevents layout shifts from affecting siblings
```

## Measuring Performance

```tsx
// app/components/web-vitals.tsx
'use client';
import { useReportWebVitals } from 'next/web-vitals';

export function WebVitals() {
  useReportWebVitals((metric) => {
    // Send to analytics
    const body = {
      name: metric.name,       // LCP, INP, CLS, FCP, TTFB
      value: metric.value,
      rating: metric.rating,   // good, needs-improvement, poor
      delta: metric.delta,
      id: metric.id,
      navigationType: metric.navigationType,
    };

    // Beacon API for reliable delivery
    if (navigator.sendBeacon) {
      navigator.sendBeacon('/api/vitals', JSON.stringify(body));
    }
  });

  return null;
}
```

## Lighthouse CI

```yaml
# .github/workflows/lighthouse.yml
- name: Lighthouse CI
  uses: treosh/lighthouse-ci-action@v12
  with:
    urls: |
      http://localhost:3000/
      http://localhost:3000/products
    budgetPath: ./lighthouse-budget.json
    uploadArtifacts: true
```

```json
// lighthouse-budget.json
[{
  "path": "/*",
  "timings": [
    { "metric": "first-contentful-paint", "budget": 2000 },
    { "metric": "largest-contentful-paint", "budget": 2500 },
    { "metric": "interactive", "budget": 3500 },
    { "metric": "total-blocking-time", "budget": 300 }
  ],
  "resourceSizes": [
    { "resourceType": "script", "budget": 300 },
    { "resourceType": "total", "budget": 500 }
  ]
}]
```
