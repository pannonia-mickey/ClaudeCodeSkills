# React 18+ Best Practices

## Concurrent Features

### useTransition

Use `useTransition` to mark state updates as non-urgent, keeping the UI responsive during expensive re-renders.

```typescript
import { useTransition, useState } from 'react';

function SearchableList({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const [filteredItems, setFilteredItems] = useState(items);
  const [isPending, startTransition] = useTransition();

  function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
    const value = e.target.value;
    setQuery(value); // Urgent: update input immediately

    startTransition(() => {
      // Non-urgent: filter can be deferred
      const filtered = items.filter((item) =>
        item.name.toLowerCase().includes(value.toLowerCase())
      );
      setFilteredItems(filtered);
    });
  }

  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {isPending && <Spinner />}
      <ItemList items={filteredItems} />
    </div>
  );
}
```

`useTransition` is ideal for tab switching, navigation, filtering large datasets, and any update where the user should not feel the UI freeze.

### useDeferredValue

Use `useDeferredValue` when you do not control the state update but want to defer a derived value's impact on rendering.

```typescript
import { useDeferredValue, useMemo } from 'react';

function ProductGrid({ searchQuery, products }: Props) {
  const deferredQuery = useDeferredValue(searchQuery);
  const isStale = searchQuery !== deferredQuery;

  const filteredProducts = useMemo(
    () => products.filter((p) => p.name.includes(deferredQuery)),
    [products, deferredQuery]
  );

  return (
    <div style={{ opacity: isStale ? 0.7 : 1, transition: 'opacity 200ms' }}>
      <ProductList products={filteredProducts} />
    </div>
  );
}
```

The stale check (`searchQuery !== deferredQuery`) allows showing a visual hint that results are updating.

## React Server Components (RSC)

### Architecture Principles

Server Components run on the server and send serialized UI to the client. They have zero impact on bundle size because their code never ships to the browser.

```typescript
// app/products/page.tsx — Server Component (default)
import { db } from '@/lib/db';
import { ProductGrid } from './ProductGrid';
import { ProductFilters } from './ProductFilters';

export default async function ProductsPage({
  searchParams,
}: {
  searchParams: Promise<{ category?: string; sort?: string }>;
}) {
  const params = await searchParams;
  const products = await db.product.findMany({
    where: params.category ? { category: params.category } : undefined,
    orderBy: { [params.sort ?? 'createdAt']: 'desc' },
  });

  return (
    <main>
      <h1>Products</h1>
      <ProductFilters /> {/* Client Component for interactivity */}
      <ProductGrid products={products} /> {/* Can be Server or Client */}
    </main>
  );
}
```

### Serialization Rules

Server Components can pass props to Client Components, but those props must be serializable: strings, numbers, booleans, arrays, plain objects, Date, Map, Set, TypedArrays, FormData, and other Server Components (as JSX). Functions, classes, and symbols cannot cross the boundary.

```typescript
// GOOD: Passing serializable data
<ClientChart data={salesData} title="Revenue" />

// BAD: Passing a function from server to client
<ClientButton onClick={() => deleteThing(id)} /> // Will not work

// GOOD: Use Server Actions instead
<ClientButton action={deleteThingAction} />
```

### Streaming SSR

React 18 supports streaming HTML with Suspense boundaries. Each boundary becomes an independent streaming chunk.

```typescript
export default function DashboardPage() {
  return (
    <div>
      <Header /> {/* Renders immediately */}
      <Suspense fallback={<MetricsSkeleton />}>
        <DashboardMetrics /> {/* Streams when ready */}
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart /> {/* Streams independently */}
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders /> {/* Streams independently */}
      </Suspense>
    </div>
  );
}
```

Each Suspense boundary allows the server to flush HTML progressively. Users see the shell immediately and content fills in as data resolves.

## TypeScript Strict Mode

### Configuration

Enable all strict checks in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

### Strict Patterns for React

Type component return values and avoid implicit `any`:

```typescript
// Use discriminated unions for variants
type AlertProps =
  | { variant: 'info'; message: string }
  | { variant: 'action'; message: string; onAction: () => void };

function Alert(props: AlertProps) {
  return (
    <div role="alert" className={styles[props.variant]}>
      <p>{props.message}</p>
      {props.variant === 'action' && (
        <button onClick={props.onAction}>Take Action</button>
      )}
    </div>
  );
}

// Type context values strictly
interface AuthContextValue {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => Promise<void>;
}

// Use const assertions for route configs
const ROUTES = {
  home: '/',
  products: '/products',
  product: (id: string) => `/products/${id}`,
} as const;
```

## Accessibility

### Semantic HTML

Always use the correct HTML element before reaching for ARIA. A `<button>` is better than a `<div role="button">`.

```typescript
// GOOD: Semantic elements
function Navigation() {
  return (
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/home">Home</a></li>
        <li><a href="/products" aria-current="page">Products</a></li>
      </ul>
    </nav>
  );
}
```

### Focus Management

Manage focus when content changes dynamically: modals, route transitions, toast notifications.

```typescript
function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const closeButtonRef = useRef<HTMLButtonElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement;
      closeButtonRef.current?.focus();
    } else {
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  // Trap focus inside modal
  // Return focus to trigger element on close
}
```

### Keyboard Navigation

All interactive elements must be operable via keyboard. Implement arrow key navigation for custom widgets.

```typescript
function TabList({ tabs, activeTab, onTabChange }: TabListProps) {
  function handleKeyDown(e: React.KeyboardEvent, index: number) {
    let nextIndex: number;
    switch (e.key) {
      case 'ArrowRight':
        nextIndex = (index + 1) % tabs.length;
        break;
      case 'ArrowLeft':
        nextIndex = (index - 1 + tabs.length) % tabs.length;
        break;
      case 'Home':
        nextIndex = 0;
        break;
      case 'End':
        nextIndex = tabs.length - 1;
        break;
      default:
        return;
    }
    e.preventDefault();
    onTabChange(tabs[nextIndex].id);
    // Focus the next tab button
  }

  return (
    <div role="tablist">
      {tabs.map((tab, index) => (
        <button
          key={tab.id}
          role="tab"
          aria-selected={tab.id === activeTab}
          tabIndex={tab.id === activeTab ? 0 : -1}
          onKeyDown={(e) => handleKeyDown(e, index)}
          onClick={() => onTabChange(tab.id)}
        >
          {tab.label}
        </button>
      ))}
    </div>
  );
}
```

### Live Regions

Announce dynamic content changes to screen readers.

```typescript
function SearchResults({ results, isLoading }: SearchResultsProps) {
  return (
    <>
      <div aria-live="polite" aria-atomic="true" className="sr-only">
        {isLoading
          ? 'Searching...'
          : `${results.length} results found`}
      </div>
      <ul>
        {results.map((result) => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </>
  );
}
```

## Performance Profiling

### React DevTools Profiler

Use the React DevTools Profiler to identify which components re-render and why. Record interactions and inspect the flame chart.

Key metrics to watch:
- **Commit duration**: Total time React spent rendering for a given commit
- **Component render time**: Time spent in each component's render
- **Why did this render?**: Enable "Record why each component rendered" in Profiler settings

### Identifying Unnecessary Re-renders

Common causes of unnecessary re-renders:

1. **Unstable references**: Objects or arrays created inline in JSX
2. **Context over-subscription**: Consuming a context that changes frequently
3. **Missing keys**: Incorrect or missing `key` props causing reconciliation issues
4. **Parent re-renders**: Parent updates causing all children to re-render

```typescript
// BAD: New object reference every render
<UserCard style={{ margin: 10 }} config={{ showAvatar: true }} />

// GOOD: Stable references
const cardStyle = { margin: 10 } as const;
const cardConfig = { showAvatar: true } as const;
<UserCard style={cardStyle} config={cardConfig} />
```

### React.memo

Apply `React.memo` only after profiling confirms a component re-renders unnecessarily with the same props.

```typescript
const ExpensiveChart = React.memo(function ExpensiveChart({ data }: ChartProps) {
  // Expensive rendering logic
  return <canvas ref={canvasRef} />;
});
```

Combine with `useMemo` for computed props and `useCallback` for function props passed to memoized children.

### Bundle Analysis

Use `@next/bundle-analyzer` or `source-map-explorer` to inspect bundle composition. Look for:
- Duplicate dependencies
- Large libraries that could be replaced or tree-shaken
- Code that should be lazy-loaded

```typescript
// Lazy load heavy components
const MarkdownEditor = lazy(() => import('./MarkdownEditor'));
const ChartLibrary = lazy(() => import('./ChartLibrary'));

// Route-based code splitting in Next.js happens automatically
// For component-level splitting, use dynamic imports
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <Skeleton />,
  ssr: false, // Skip SSR if component uses browser-only APIs
});
```

### Web Vitals

Monitor Core Web Vitals (LCP, FID/INP, CLS) in production:

```typescript
// Next.js built-in reporting
export function reportWebVitals(metric: NextWebVitalsMetric) {
  switch (metric.name) {
    case 'LCP':
    case 'INP':
    case 'CLS':
      analytics.track('web-vital', {
        name: metric.name,
        value: metric.value,
        rating: metric.rating,
      });
      break;
  }
}
```

Set performance budgets: LCP under 2.5 seconds, INP under 200 milliseconds, CLS under 0.1. Profile with Chrome DevTools Performance panel using CPU throttling to simulate real user conditions.
