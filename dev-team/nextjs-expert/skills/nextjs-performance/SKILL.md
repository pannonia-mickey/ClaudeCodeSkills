---
name: Next.js Performance
description: This skill should be used when the user asks about "next/image", "next/font", "Next.js bundle size", "Next.js code splitting", "next/dynamic", "Partial Prerendering", "streaming SSR", "Core Web Vitals", "Lighthouse score", or "Next.js performance optimization". It covers image optimization, font loading, code splitting, bundle analysis, and performance budgets.
---

# Next.js Performance

## Image Optimization

```tsx
import Image from 'next/image';

// Local image (auto-sized)
import heroImage from '@/public/hero.jpg';

export function Hero() {
  return (
    <Image
      src={heroImage}
      alt="Hero banner"
      priority          // Preload for LCP image
      placeholder="blur" // Blur-up effect (auto for local images)
      sizes="100vw"
    />
  );
}

// Remote image
export function ProductImage({ src, alt }: { src: string; alt: string }) {
  return (
    <Image
      src={src}
      alt={alt}
      width={400}
      height={300}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      loading="lazy"    // Default — lazy load below fold
      quality={80}      // Default 75, adjust per use case
    />
  );
}

// Fill mode for unknown dimensions
export function Avatar({ src, alt }: { src: string; alt: string }) {
  return (
    <div className="relative h-12 w-12 overflow-hidden rounded-full">
      <Image src={src} alt={alt} fill className="object-cover" sizes="48px" />
    </div>
  );
}
```

## Font Optimization

```tsx
// app/layout.tsx — zero CLS font loading
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter', // CSS variable for Tailwind
});

const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-roboto-mono',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${robotoMono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}

// tailwind.config.ts
// fontFamily: {
//   sans: ['var(--font-inter)', ...defaultTheme.fontFamily.sans],
//   mono: ['var(--font-roboto-mono)', ...defaultTheme.fontFamily.mono],
// }

// Local font
import localFont from 'next/font/local';
const myFont = localFont({ src: './fonts/MyFont.woff2', display: 'swap' });
```

## Code Splitting with next/dynamic

```tsx
import dynamic from 'next/dynamic';

// Lazy load heavy components
const Chart = dynamic(() => import('@/components/chart'), {
  loading: () => <div className="h-64 animate-pulse bg-gray-200" />,
  ssr: false, // Client-only (no SSR for browser-dependent libraries)
});

const RichTextEditor = dynamic(() => import('@/components/editor'), {
  loading: () => <textarea className="h-48 w-full" />,
  ssr: false,
});

// Named export
const Tab = dynamic(() =>
  import('@/components/tabs').then((mod) => mod.Tab)
);

// Conditional loading
export function Dashboard({ showChart }: { showChart: boolean }) {
  return (
    <div>
      <h1>Dashboard</h1>
      {showChart && <Chart data={data} />}
    </div>
  );
}
```

## Bundle Analysis

```bash
# Install analyzer
npm install @next/bundle-analyzer

# next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer(nextConfig);

# Run analysis
ANALYZE=true npm run build

# Common optimizations:
# 1. Replace moment.js with date-fns (tree-shakeable)
# 2. Dynamic import heavy libraries (chart.js, monaco-editor)
# 3. Use next/dynamic for components not needed on initial load
# 4. Check for duplicate packages in bundle
# 5. Use modularizeImports for icon libraries
```

```javascript
// next.config.js — modularize imports
module.exports = {
  modularizeImports: {
    'lucide-react': {
      transform: 'lucide-react/dist/esm/icons/{{kebabCase member}}',
    },
    'lodash': {
      transform: 'lodash/{{member}}',
    },
  },
};
```

## Partial Prerendering (PPR)

```tsx
// next.config.js
// experimental: { ppr: true }

// Combines static shell with dynamic content
// Static parts served instantly from cache
// Dynamic parts stream in via Suspense

export default function ProductPage() {
  return (
    <div>
      {/* Static — pre-rendered at build time */}
      <Header />
      <ProductInfo />
      <Footer />

      {/* Dynamic — streamed at request time */}
      <Suspense fallback={<PriceSkeleton />}>
        <DynamicPrice />  {/* Uses cookies() for geo-pricing */}
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <UserReviews />   {/* Personalized content */}
      </Suspense>
    </div>
  );
}
```

## References

- [Web Vitals Optimization](references/web-vitals.md) — LCP, FID/INP, CLS optimization strategies, measuring performance, Lighthouse CI.
- [Rendering Strategies](references/rendering-strategies.md) — SSG vs SSR vs ISR vs CSR decision matrix, hybrid approaches, edge rendering.
