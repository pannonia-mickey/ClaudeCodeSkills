# Routing Patterns

## Modal with Intercepting Routes

```tsx
// Instagram-style: click photo → modal over feed, direct URL → full page

// app/@modal/(.)photo/[id]/page.tsx — intercepted route (modal)
import { Modal } from '@/components/modal';

export default async function PhotoModal({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const photo = await getPhoto(id);

  return (
    <Modal>
      <img src={photo.url} alt={photo.alt} />
      <p>{photo.caption}</p>
    </Modal>
  );
}

// components/modal.tsx
'use client';
import { useRouter } from 'next/navigation';

export function Modal({ children }: { children: React.ReactNode }) {
  const router = useRouter();

  return (
    <div className="fixed inset-0 z-50 bg-black/50" onClick={() => router.back()}>
      <div className="mx-auto mt-20 max-w-lg rounded bg-white p-6" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}

// app/photo/[id]/page.tsx — direct URL (full page)
export default async function PhotoPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const photo = await getPhoto(id);
  return <div className="max-w-4xl mx-auto">{/* Full page layout */}</div>;
}
```

## Search Params with URL State

```tsx
// Sync filters/pagination with URL for shareability and back-button support

// app/products/page.tsx (Server Component)
export default async function ProductsPage({
  searchParams,
}: {
  searchParams: Promise<{ q?: string; page?: string; sort?: string }>;
}) {
  const params = await searchParams;
  const query = params.q || '';
  const page = parseInt(params.page || '1');
  const sort = params.sort || 'newest';

  const products = await getProducts({ query, page, sort });

  return (
    <div>
      <SearchFilters query={query} sort={sort} />
      <ProductGrid products={products.data} />
      <Pagination current={page} total={products.totalPages} />
    </div>
  );
}

// components/search-filters.tsx
'use client';
import { useRouter, useSearchParams, usePathname } from 'next/navigation';
import { useCallback } from 'react';

export function SearchFilters({ query, sort }: { query: string; sort: string }) {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const updateParams = useCallback(
    (key: string, value: string) => {
      const params = new URLSearchParams(searchParams.toString());
      if (value) params.set(key, value);
      else params.delete(key);
      params.delete('page'); // Reset page on filter change
      router.push(`${pathname}?${params.toString()}`);
    },
    [searchParams, pathname, router]
  );

  return (
    <div className="flex gap-4">
      <input
        defaultValue={query}
        onChange={(e) => updateParams('q', e.target.value)}
        placeholder="Search..."
      />
      <select
        defaultValue={sort}
        onChange={(e) => updateParams('sort', e.target.value)}
      >
        <option value="newest">Newest</option>
        <option value="price-asc">Price: Low to High</option>
        <option value="price-desc">Price: High to Low</option>
      </select>
    </div>
  );
}
```

## Breadcrumbs from Route Segments

```tsx
// components/breadcrumbs.tsx
'use client';
import { usePathname } from 'next/navigation';
import Link from 'next/link';

const labels: Record<string, string> = {
  dashboard: 'Dashboard',
  settings: 'Settings',
  profile: 'Profile',
  orders: 'Orders',
};

export function Breadcrumbs() {
  const pathname = usePathname();
  const segments = pathname.split('/').filter(Boolean);

  return (
    <nav aria-label="Breadcrumb">
      <ol className="flex gap-2">
        <li><Link href="/">Home</Link></li>
        {segments.map((segment, i) => {
          const href = '/' + segments.slice(0, i + 1).join('/');
          const isLast = i === segments.length - 1;
          const label = labels[segment] || segment;

          return (
            <li key={href} className="flex items-center gap-2">
              <span>/</span>
              {isLast ? (
                <span aria-current="page">{label}</span>
              ) : (
                <Link href={href}>{label}</Link>
              )}
            </li>
          );
        })}
      </ol>
    </nav>
  );
}
```

## Pagination Component

```tsx
// components/pagination.tsx
import Link from 'next/link';

export function Pagination({
  current,
  total,
  searchParams,
}: {
  current: number;
  total: number;
  searchParams: Record<string, string>;
}) {
  const createPageUrl = (page: number) => {
    const params = new URLSearchParams(searchParams);
    params.set('page', String(page));
    return `?${params.toString()}`;
  };

  return (
    <nav aria-label="Pagination" className="flex gap-2">
      {current > 1 && (
        <Link href={createPageUrl(current - 1)}>Previous</Link>
      )}
      {Array.from({ length: total }, (_, i) => i + 1).map((page) => (
        <Link
          key={page}
          href={createPageUrl(page)}
          aria-current={page === current ? 'page' : undefined}
          className={page === current ? 'font-bold' : ''}
        >
          {page}
        </Link>
      ))}
      {current < total && (
        <Link href={createPageUrl(current + 1)}>Next</Link>
      )}
    </nav>
  );
}
```
