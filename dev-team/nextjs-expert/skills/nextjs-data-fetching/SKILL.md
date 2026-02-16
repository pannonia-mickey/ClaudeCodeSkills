---
name: Next.js Data Fetching
description: This skill should be used when the user asks about "Next.js data fetching", "fetch caching", "revalidation", "ISR", "SSG", "SSR", "Server Actions", "generateStaticParams", "React Query with Next.js", "on-demand revalidation", or "streaming SSR". It covers Next.js data fetching patterns, caching, revalidation, and Server Actions.
---

# Next.js Data Fetching

## Server Component Data Fetching

```tsx
// app/products/page.tsx — Server Component (default)
// Data fetches directly in component — no useEffect needed

// Cached (default) — equivalent to SSG/ISR
async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    next: { revalidate: 3600 }, // Revalidate every hour
  });
  if (!res.ok) throw new Error('Failed to fetch products');
  return res.json();
}

// Dynamic (no cache) — equivalent to SSR
async function getCurrentUser() {
  const res = await fetch('https://api.example.com/me', {
    cache: 'no-store', // Always fresh
    headers: { Authorization: `Bearer ${getToken()}` },
  });
  return res.json();
}

export default async function ProductsPage() {
  const products = await getProducts();  // Cached
  const user = await getCurrentUser();   // Dynamic

  return <ProductGrid products={products} user={user} />;
}
```

## Static Generation with generateStaticParams

```tsx
// app/blog/[slug]/page.tsx
// Pre-render at build time
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

// Optionally handle unknown slugs
export const dynamicParams = true; // false = return 404 for unknown

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  if (!post) notFound();
  return <article>{post.content}</article>;
}
```

## Streaming with Suspense

```tsx
// app/dashboard/page.tsx — stream independent sections
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div className="grid grid-cols-3 gap-4">
      {/* Each section loads independently */}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsSection />
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />
      </Suspense>
      <Suspense fallback={<ListSkeleton />}>
        <RecentOrders />
      </Suspense>
    </div>
  );
}

// Each is an async Server Component
async function StatsSection() {
  const stats = await getStats(); // 200ms
  return <StatsGrid stats={stats} />;
}

async function RevenueChart() {
  const revenue = await getRevenue(); // 500ms
  return <Chart data={revenue} />;
}

async function RecentOrders() {
  const orders = await getRecentOrders(); // 300ms
  return <OrderList orders={orders} />;
}

// Result: Stats appears at 200ms, Orders at 300ms, Chart at 500ms
// Instead of everything waiting 500ms
```

## Server Actions

```tsx
// actions/todo.ts
'use server';

import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const createTodoSchema = z.object({
  title: z.string().min(1).max(200),
});

export async function createTodo(prevState: unknown, formData: FormData) {
  const parsed = createTodoSchema.safeParse({
    title: formData.get('title'),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  await db.todo.create({ data: { title: parsed.data.title } });
  revalidatePath('/todos');
  return { success: true };
}

export async function deleteTodo(id: string) {
  await db.todo.delete({ where: { id } });
  revalidatePath('/todos');
}

// components/todo-form.tsx
'use client';
import { useActionState, useOptimistic } from 'react';
import { createTodo } from '@/actions/todo';

export function TodoForm({ todos }: { todos: Todo[] }) {
  const [state, action, pending] = useActionState(createTodo, null);
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo: string) => [...state, { id: 'temp', title: newTodo, completed: false }]
  );

  return (
    <>
      <form action={(formData) => {
        addOptimistic(formData.get('title') as string);
        action(formData);
      }}>
        <input name="title" required />
        {state?.error?.title && <p className="text-red-500">{state.error.title}</p>}
        <button disabled={pending}>
          {pending ? 'Adding...' : 'Add Todo'}
        </button>
      </form>
      <ul>
        {optimisticTodos.map((todo) => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </>
  );
}
```

## On-Demand Revalidation

```tsx
// Revalidate by tag
// In data fetching:
const products = await fetch('https://api.example.com/products', {
  next: { tags: ['products'] },
});

// In Server Action or Route Handler:
import { revalidateTag, revalidatePath } from 'next/cache';

export async function updateProduct(id: string, data: ProductData) {
  await db.product.update({ where: { id }, data });
  revalidateTag('products');      // Revalidate all fetches tagged 'products'
  revalidatePath('/products');     // Revalidate specific path
  revalidatePath('/products/[id]', 'page'); // Revalidate dynamic page
}

// Webhook Route Handler for external revalidation
// app/api/revalidate/route.ts
export async function POST(request: NextRequest) {
  const secret = request.headers.get('x-revalidation-secret');
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { tag } = await request.json();
  revalidateTag(tag);
  return NextResponse.json({ revalidated: true });
}
```

## References

- [Caching Deep Dive](references/caching-deep-dive.md) — Request memoization, data cache, full route cache, router cache, cache interactions.
- [Server Actions Patterns](references/server-actions-patterns.md) — Form patterns, file uploads, optimistic updates, error handling, progressive enhancement.
