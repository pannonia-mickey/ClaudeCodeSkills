# Project Patterns

## Recommended Folder Structure

```
src/
├── app/                     # Next.js App Router
│   ├── (marketing)/         # Public pages group
│   │   ├── layout.tsx
│   │   ├── page.tsx         # Landing page
│   │   ├── about/page.tsx
│   │   └── pricing/page.tsx
│   ├── (app)/               # Authenticated app group
│   │   ├── layout.tsx       # With auth check
│   │   ├── dashboard/
│   │   │   ├── page.tsx
│   │   │   ├── loading.tsx
│   │   │   └── error.tsx
│   │   └── settings/
│   │       └── page.tsx
│   ├── api/                 # Route Handlers
│   │   ├── auth/[...nextauth]/route.ts
│   │   └── webhooks/stripe/route.ts
│   ├── layout.tsx           # Root layout
│   └── not-found.tsx
├── components/              # Shared components
│   ├── ui/                  # Primitives (Button, Input, Card)
│   └── features/            # Feature-specific (UserAvatar, PricingCard)
├── lib/                     # Utilities and configs
│   ├── db.ts                # Database client
│   ├── auth.ts              # Auth configuration
│   └── utils.ts             # Shared utilities
├── actions/                 # Server Actions
│   ├── user.ts
│   └── order.ts
└── types/                   # TypeScript types
    └── index.ts
```

## Route Groups

```tsx
// Use route groups to:
// 1. Organize without affecting URLs
// 2. Apply different layouts to different sections
// 3. Create multiple root layouts

// (marketing) — public layout with nav + footer
// app/(marketing)/layout.tsx
export default function MarketingLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <MarketingNav />
      {children}
      <Footer />
    </>
  );
}

// (app) — authenticated layout with sidebar
// app/(app)/layout.tsx
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const session = await auth();
  if (!session) redirect('/login');

  return (
    <div className="flex">
      <Sidebar user={session.user} />
      <main className="flex-1 p-6">{children}</main>
    </div>
  );
}
```

## Colocation Pattern

```
// Colocate related files with their routes
app/dashboard/
├── page.tsx                # Route component
├── loading.tsx             # Loading state
├── error.tsx               # Error boundary
├── _components/            # Route-specific components (underscore = private)
│   ├── stats-card.tsx
│   ├── recent-orders.tsx
│   └── activity-chart.tsx
├── _actions/               # Route-specific Server Actions
│   └── refresh-stats.ts
└── _lib/                   # Route-specific utilities
    └── format-stats.ts

// Underscore prefix (_) prevents Next.js from treating as routes
// Keeps related code close to where it's used
```

## Shared Layouts with Slots

```tsx
// Parallel routes for complex layouts
// app/(app)/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  stats,
  activity,
}: {
  children: React.ReactNode;
  stats: React.ReactNode;
  activity: React.ReactNode;
}) {
  return (
    <div>
      <div className="grid grid-cols-3 gap-4">{stats}</div>
      <div className="grid grid-cols-2 gap-4">
        <div>{children}</div>
        <div>{activity}</div>
      </div>
    </div>
  );
}

// app/(app)/dashboard/@stats/page.tsx — stats slot
// app/(app)/dashboard/@activity/page.tsx — activity slot
// Each slot loads independently with its own loading/error states
```

## Server/Client Component Boundary

```tsx
// Rule: Push "use client" as far down as possible

// BAD — entire page is client
'use client';
export default function ProductPage() {
  const [count, setCount] = useState(0);
  const product = useProduct(); // Client-side fetch
  return <div>...</div>;
}

// GOOD — server page with client island
// app/products/[id]/page.tsx (Server Component)
export default async function ProductPage({ params }: Props) {
  const { id } = await params;
  const product = await getProduct(id); // Server-side fetch
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartButton productId={product.id} /> {/* Client Component */}
    </div>
  );
}

// components/add-to-cart-button.tsx
'use client';
export function AddToCartButton({ productId }: { productId: string }) {
  const [count, setCount] = useState(1);
  return <button onClick={() => addToCart(productId, count)}>Add to Cart</button>;
}
```
