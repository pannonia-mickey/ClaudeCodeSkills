---
name: SEO Performance Optimization
description: This skill should be used when the user asks to "improve Core Web Vitals", "fix LCP", "fix CLS", "fix INP", "optimize page speed for SEO", "reduce render-blocking resources", "optimize images for SEO", "improve Lighthouse score", or mentions performance metrics affecting search ranking like TTFB, FCP, or page load time.
---

## Purpose

Provide diagnostic and implementation patterns for optimizing Core Web Vitals and page performance metrics that directly influence Google's search ranking algorithm. Since 2021, Core Web Vitals are a confirmed ranking signal — pages that meet Google's thresholds have a measurable advantage.

## Core Web Vitals Thresholds

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | ≤ 2.5s | 2.5s – 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | ≤ 200ms | 200ms – 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | ≤ 0.1 | 0.1 – 0.25 | > 0.25 |

These thresholds apply to the 75th percentile of page loads (p75), measured from real user data (CrUX).

## LCP Optimization

### Identifying the LCP Element

The LCP element is the largest image, video, or text block visible in the viewport when the page loads. Common LCP elements:

- Hero images or banner images
- `<h1>` text blocks with large font sizes
- Background images applied via CSS
- `<video>` poster images

### Optimization Strategies

#### 1. Preload the LCP Resource

```html
<!-- Preload hero image -->
<link rel="preload" as="image" href="/images/hero.webp"
      fetchpriority="high" type="image/webp" />

<!-- Preload with responsive images -->
<link rel="preload" as="image" href="/images/hero.webp"
      imagesrcset="/images/hero-400.webp 400w,
                   /images/hero-800.webp 800w,
                   /images/hero-1200.webp 1200w"
      imagesizes="100vw" />
```

#### 2. Optimize Image Delivery

```html
<!-- Responsive images with modern formats -->
<picture>
  <source type="image/avif"
          srcset="/img/hero-400.avif 400w,
                 /img/hero-800.avif 800w,
                 /img/hero-1200.avif 1200w"
          sizes="100vw" />
  <source type="image/webp"
          srcset="/img/hero-400.webp 400w,
                 /img/hero-800.webp 800w,
                 /img/hero-1200.webp 1200w"
          sizes="100vw" />
  <img src="/img/hero-800.jpg"
       alt="Product showcase"
       width="1200" height="630"
       fetchpriority="high"
       decoding="async" />
</picture>
```

**Key attributes:**
- `fetchpriority="high"` on the LCP image (only one per page).
- `width` and `height` to reserve space and prevent CLS.
- `decoding="async"` to avoid blocking the main thread.
- Do NOT use `loading="lazy"` on the LCP image.

#### 3. Reduce Server Response Time (TTFB)

- Enable server-side caching (CDN, reverse proxy, application cache).
- Use streaming SSR where supported (React 18 `renderToPipeableStream`, Next.js streaming).
- Optimize database queries behind the critical rendering path.
- Target TTFB under 200ms for the document HTML.

#### 4. Eliminate Render-Blocking Resources

```html
<!-- Inline critical CSS -->
<style>
  /* Critical above-the-fold styles inlined here */
  .hero { ... }
  .nav { ... }
</style>

<!-- Defer non-critical CSS -->
<link rel="preload" href="/css/styles.css" as="style"
      onload="this.onload=null;this.rel='stylesheet'" />
<noscript><link rel="stylesheet" href="/css/styles.css" /></noscript>

<!-- Defer JavaScript -->
<script src="/js/app.js" defer></script>
```

## INP Optimization

### Diagnosis

INP measures the latency of the slowest interaction (click, tap, key press) during the page visit. Poor INP is caused by:

- Long JavaScript tasks blocking the main thread during interaction.
- Large DOM size causing slow style recalculation and layout.
- Synchronous XHR or heavy computation in event handlers.

### Optimization Strategies

#### 1. Break Up Long Tasks

```javascript
// Bad: single long task blocks interaction
function processLargeList(items) {
  items.forEach(item => heavyComputation(item));
}

// Good: yield to main thread between chunks
async function processLargeList(items) {
  const CHUNK_SIZE = 50;
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    chunk.forEach(item => heavyComputation(item));
    await new Promise(resolve => setTimeout(resolve, 0)); // yield
  }
}
```

#### 2. Use requestIdleCallback for Non-Critical Work

```javascript
// Defer analytics and tracking to idle time
requestIdleCallback(() => {
  initAnalytics();
  trackPageView();
});
```

#### 3. Reduce DOM Size

Target fewer than 1,400 DOM elements. Virtualize long lists and tables. Avoid deeply nested DOM trees (max depth ~32 levels).

## CLS Optimization

### Common Causes and Fixes

#### 1. Images Without Dimensions

```html
<!-- Bad: no dimensions — browser cannot reserve space -->
<img src="photo.jpg" alt="Product" />

<!-- Good: explicit dimensions prevent layout shift -->
<img src="photo.jpg" alt="Product" width="800" height="600" />

<!-- Good: CSS aspect-ratio for responsive images -->
<style>
  .responsive-img {
    aspect-ratio: 16 / 9;
    width: 100%;
    height: auto;
  }
</style>
```

#### 2. Web Font Layout Shifts (FOIT/FOUT)

```css
/* Prevent invisible text flash */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
  /* optional: use size-adjust to match fallback metrics */
  size-adjust: 105%;
  ascent-override: 90%;
  descent-override: 20%;
  line-gap-override: 0%;
}
```

```html
<!-- Preconnect to font CDN -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Preload critical font file -->
<link rel="preload" as="font" type="font/woff2"
      href="/fonts/custom.woff2" crossorigin />
```

#### 3. Dynamically Injected Content

- Reserve space for ad slots, cookie banners, and notification bars with CSS `min-height`.
- Use CSS `contain: layout` on containers that will have content injected.
- Load above-the-fold dynamic content server-side when possible.

#### 4. CSS Containment for Performance

```css
/* Isolate layout calculations to specific containers */
.product-card {
  contain: layout style paint;
  content-visibility: auto;
  contain-intrinsic-size: 0 300px;
}
```

`content-visibility: auto` skips rendering off-screen elements, significantly improving initial render for long pages.

## Framework-Specific SSR/SSG for SEO

### Next.js

```typescript
// Static generation (best for SEO — fastest TTFB)
export async function generateStaticParams() {
  const products = await getProducts();
  return products.map(p => ({ slug: p.slug }));
}

// ISR for frequently updated content
export const revalidate = 3600; // revalidate every hour

// Streaming SSR for dynamic content
import { Suspense } from 'react';

export default function Page() {
  return (
    <>
      <StaticHero />  {/* Renders immediately */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <DynamicReviews />  {/* Streams when ready */}
      </Suspense>
    </>
  );
}
```

### Rendering Strategy Decision

| Strategy | TTFB | SEO | Dynamic Data | Use When |
|----------|------|-----|-------------|----------|
| SSG | Fastest | Best | No | Content changes infrequently |
| ISR | Fast | Great | Periodic | Content updates hourly/daily |
| SSR | Moderate | Great | Real-time | Content changes per request |
| CSR | Slow* | Poor | Real-time | Authenticated dashboards only |

*CSR requires JavaScript execution for content — search engines may not render it.

## Measurement and Monitoring

### Lab Tools (Development)

- **Lighthouse**: Built into Chrome DevTools, measures simulated performance.
- **WebPageTest**: Real browser testing with filmstrip view and waterfall analysis.
- **Chrome DevTools Performance tab**: Flame chart for identifying long tasks and layout shifts.

### Field Data (Production)

- **Google Search Console Core Web Vitals report**: Real user data from CrUX, grouped by page type.
- **CrUX Dashboard**: BigQuery dataset of real user performance data.
- **web-vitals library**: Measure CWV in production with custom reporting.

```javascript
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(metric => sendToAnalytics('LCP', metric));
onINP(metric => sendToAnalytics('INP', metric));
onCLS(metric => sendToAnalytics('CLS', metric));
```

## References

- **[core-web-vitals-deep-dive.md](references/core-web-vitals-deep-dive.md)** — Advanced CWV optimization including LCP sub-parts analysis (TTFB, resource load delay, resource load time, element render delay), INP debugging with Long Animation Frames API, CLS attribution with Layout Instability API, and performance budgets for different page types.
- **[resource-loading-strategies.md](references/resource-loading-strategies.md)** — Comprehensive resource loading guide covering preload/prefetch/preconnect strategies, script loading patterns (async vs defer vs module), CSS delivery optimization (critical CSS extraction, code splitting), image CDN configuration, HTTP/2 and HTTP/3 multiplexing benefits, and caching strategies (Cache-Control, stale-while-revalidate, service workers).
