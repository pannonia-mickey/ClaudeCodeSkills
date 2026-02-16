---
name: Next.js Mastery
description: This skill should be used when the user asks about "Next.js setup", "App Router", "file-based routing", "Next.js layouts", "Next.js loading states", "Next.js error handling", "Next.js metadata", "next.config.js", or "Next.js project structure". It covers core Next.js concepts, App Router fundamentals, and project configuration.
---

# Next.js Mastery

## App Router File Conventions

```
app/
├── layout.tsx          # Root layout (required)
├── page.tsx            # Home page (/)
├── loading.tsx         # Loading UI (Suspense boundary)
├── error.tsx           # Error boundary
├── not-found.tsx       # 404 page
├── global-error.tsx    # Root error boundary
├── (marketing)/        # Route group (no URL segment)
│   ├── layout.tsx
│   ├── about/page.tsx  # /about
│   └── blog/page.tsx   # /blog
├── (app)/              # Route group
│   ├── layout.tsx      # Authenticated layout
│   ├── dashboard/
│   │   ├── page.tsx    # /dashboard
│   │   └── loading.tsx
│   └── settings/
│       └── page.tsx    # /settings
├── api/
│   └── users/
│       └── route.ts    # API Route Handler
└── [...slug]/
    └── page.tsx        # Catch-all route
```

## Root Layout

```tsx
// app/layout.tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'], display: 'swap' });

export const metadata: Metadata = {
  title: { template: '%s | My App', default: 'My App' },
  description: 'Production Next.js application',
  metadataBase: new URL('https://example.com'),
  openGraph: { type: 'website', locale: 'en_US' },
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.className}>
      <body>
        <header><nav>{/* Navigation */}</nav></header>
        <main>{children}</main>
        <footer>{/* Footer */}</footer>
      </body>
    </html>
  );
}
```

## Loading and Error States

```tsx
// app/dashboard/loading.tsx — automatic Suspense boundary
export default function DashboardLoading() {
  return (
    <div className="grid grid-cols-3 gap-4">
      <div className="h-32 animate-pulse rounded bg-gray-200" />
      <div className="h-32 animate-pulse rounded bg-gray-200" />
      <div className="h-32 animate-pulse rounded bg-gray-200" />
    </div>
  );
}

// app/dashboard/error.tsx — error boundary
'use client';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

## Dynamic Metadata

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

type Props = { params: Promise<{ slug: string }> };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.coverImage }],
    },
  };
}

export default async function BlogPost({ params }: Props) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{/* ... */}</article>;
}
```

## next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    typedRoutes: true,     // Type-safe <Link> hrefs
    ppr: true,             // Partial Prerendering
  },
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.example.com' },
      { protocol: 'https', hostname: '*.cloudfront.net' },
    ],
  },
  // Redirect and rewrite rules
  async redirects() {
    return [
      { source: '/old-page', destination: '/new-page', permanent: true },
    ];
  },
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        ],
      },
    ];
  },
};
module.exports = nextConfig;
```

## References

- [Project Patterns](references/project-patterns.md) — Folder organization, shared layouts, route groups, colocation, barrel files.
- [Configuration Guide](references/configuration-guide.md) — Environment variables, next.config.js options, TypeScript paths, turbopack, output modes.
