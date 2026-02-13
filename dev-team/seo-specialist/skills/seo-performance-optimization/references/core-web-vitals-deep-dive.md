# Core Web Vitals Deep Dive

## LCP Sub-Parts Analysis

LCP is composed of four sub-parts. Optimize each to reduce total LCP time:

```
|-- TTFB --|-- Resource Load Delay --|-- Resource Load Time --|-- Element Render Delay --|
|   Server  |   Discovery to start    |   Download time        |   Render after download  |
```

### 1. Time to First Byte (TTFB)

Time from navigation start to receiving the first byte of the HTML document.

**Target:** < 800ms (ideally < 200ms)

**Optimization:**
- Use a CDN for geographic distribution.
- Enable server-side caching (page cache, object cache).
- Optimize database queries on the critical path.
- Use HTTP/2 or HTTP/3 for connection multiplexing.
- Enable compression (Brotli preferred over gzip).
- Use streaming SSR to send initial HTML faster.

**Measurement:**
```javascript
// PerformanceNavigationTiming API
const navEntry = performance.getEntriesByType('navigation')[0];
const ttfb = navEntry.responseStart - navEntry.requestStart;
```

### 2. Resource Load Delay

Time between TTFB and the browser starting to download the LCP resource.

**Optimization:**
- **Preload the LCP image**: `<link rel="preload" as="image">` in the HTML `<head>`.
- **Avoid lazy-loading the LCP image**: Remove `loading="lazy"` from above-the-fold images.
- **Inline critical CSS**: Prevent CSS from blocking LCP image discovery.
- **Set fetchpriority="high"**: On the LCP `<img>` element.
- **Avoid CSS background images for LCP**: Browser cannot discover them until CSS is parsed.

### 3. Resource Load Time

Time to download the LCP resource (image, video, font).

**Optimization:**
- **Serve optimized image formats**: AVIF > WebP > JPEG. Use `<picture>` with multiple sources.
- **Serve correctly sized images**: Use `srcset` and `sizes` to avoid downloading oversized images.
- **Use a CDN**: Serve assets from edge locations near the user.
- **Enable HTTP/2**: Allows multiplexed downloads without head-of-line blocking.
- **Set long cache headers**: `Cache-Control: public, max-age=31536000, immutable` for versioned assets.

### 4. Element Render Delay

Time between resource download completion and the element being rendered.

**Optimization:**
- **Remove render-blocking JavaScript**: Use `defer` or `async` on `<script>` tags.
- **Inline critical CSS**: Avoid full stylesheet blocking rendering.
- **Reduce DOM size**: Large DOMs increase style calculation and layout time.
- **Avoid `display: none` on LCP element**: Conditionally hidden elements delay rendering.
- **Minimize CSS complexity**: Reduce selector complexity and unused CSS.

## INP Debugging with Long Animation Frames API

### Long Animation Frames (LoAF) API

Provides detailed breakdown of long frames that block interaction:

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) { // Frames longer than 50ms
      console.log('Long Animation Frame:', {
        duration: entry.duration,
        blockingDuration: entry.blockingDuration,
        renderStart: entry.renderStart,
        styleAndLayoutStart: entry.styleAndLayoutStart,
        scripts: entry.scripts.map(s => ({
          sourceURL: s.sourceURL,
          sourceFunctionName: s.sourceFunctionName,
          duration: s.duration,
          executionStart: s.executionStart,
          invokerType: s.invokerType,
        })),
      });
    }
  }
});

observer.observe({ type: 'long-animation-frame', buffered: true });
```

### INP Breakdown

INP consists of three phases:

```
|-- Input Delay --|-- Processing Time --|-- Presentation Delay --|
|  Blocked thread  |  Event handler run  |  Style/Layout/Paint    |
```

1. **Input delay**: Time from user interaction to event handler start. Caused by long tasks running on the main thread when the user interacts.
2. **Processing time**: Time to execute all event handlers. Reduce by breaking up long handlers.
3. **Presentation delay**: Time from handler completion to next paint. Caused by large DOM updates, complex CSS, or forced layout.

### Reducing Input Delay

```javascript
// Bad: heavy computation on main thread during page load
loadProducts().then(products => {
  products.forEach(p => renderProduct(p)); // blocks main thread
});

// Good: use requestIdleCallback for non-critical work
loadProducts().then(products => {
  requestIdleCallback(() => {
    products.forEach(p => renderProduct(p));
  });
});

// Good: use scheduler.yield() to break up work (Chrome 129+)
async function processItems(items) {
  for (const item of items) {
    processItem(item);
    if (navigator.scheduling?.isInputPending?.()) {
      await scheduler.yield();
    }
  }
}
```

### Reducing Processing Time

```javascript
// Bad: synchronous DOM manipulation in handler
button.addEventListener('click', () => {
  updateLargeList(items);          // 200ms DOM manipulation
  recalculateLayout();             // 100ms layout
  sendAnalytics();                 // 50ms network call
});

// Good: minimal sync work, defer the rest
button.addEventListener('click', () => {
  requestAnimationFrame(() => updateLargeList(items)); // deferred
  queueMicrotask(() => sendAnalytics());              // deferred
  showLoadingIndicator();                              // instant feedback
});
```

## CLS Attribution with Layout Instability API

### Identifying CLS Sources

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) { // Exclude user-initiated shifts
      console.log('Layout shift:', {
        value: entry.value,
        sources: entry.sources?.map(s => ({
          node: s.node,
          currentRect: s.currentRect,
          previousRect: s.previousRect,
        })),
      });

      // Identify which element shifted
      entry.sources?.forEach(source => {
        if (source.node) {
          console.log('Shifted element:', source.node);
          source.node.style.outline = '3px solid red'; // Visual debug
        }
      });
    }
  }
});

observer.observe({ type: 'layout-shift', buffered: true });
```

### CLS Session Windows

CLS is measured using session windows:
- A session window starts with the first layout shift.
- The window ends after 1 second with no shifts, or 5 seconds total.
- CLS score is the maximum session window value.

### Common CLS Patterns and Fixes

#### Dynamic Ad Slots

```css
/* Reserve space for ad containers */
.ad-slot-leaderboard {
  min-height: 90px;  /* Standard leaderboard size */
  width: 100%;
}

.ad-slot-medium-rect {
  min-height: 250px; /* Standard MREC size */
  width: 300px;
}
```

#### Late-Loading Embeds

```html
<!-- Reserve space for YouTube embed -->
<div style="aspect-ratio: 16 / 9; width: 100%; background: #000;">
  <iframe
    src="https://www.youtube.com/embed/VIDEO_ID"
    width="100%" height="100%"
    loading="lazy"
    title="Video title"
  ></iframe>
</div>
```

#### Cookie Consent Banners

```css
/* Pin to bottom — does not shift content */
.cookie-banner {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 1000;
}

/* If banner is at top, reserve space from initial render */
.cookie-banner-top {
  position: sticky;
  top: 0;
  min-height: 60px;
}
```

## Performance Budgets

### Budget by Page Type

| Page Type | LCP | INP | CLS | Total JS | Total CSS | Total Images |
|-----------|-----|-----|-----|----------|-----------|-------------|
| Homepage | 2.0s | 150ms | 0.05 | 200KB | 50KB | 500KB |
| Product page | 2.5s | 200ms | 0.1 | 250KB | 60KB | 800KB |
| Category page | 2.5s | 200ms | 0.05 | 200KB | 50KB | 1MB |
| Blog article | 2.0s | 100ms | 0.05 | 150KB | 40KB | 600KB |
| Checkout | 2.0s | 150ms | 0.02 | 200KB | 50KB | 200KB |

All sizes refer to transferred (compressed) bytes.

### Budget Enforcement

```javascript
// performance-budget.json for Lighthouse CI
{
  "budgets": [
    {
      "resourceSizes": [
        { "resourceType": "script", "budget": 250 },
        { "resourceType": "stylesheet", "budget": 60 },
        { "resourceType": "image", "budget": 800 },
        { "resourceType": "font", "budget": 100 },
        { "resourceType": "total", "budget": 1500 }
      ],
      "resourceCounts": [
        { "resourceType": "script", "budget": 15 },
        { "resourceType": "stylesheet", "budget": 5 },
        { "resourceType": "font", "budget": 4 }
      ],
      "timings": [
        { "metric": "largest-contentful-paint", "budget": 2500 },
        { "metric": "cumulative-layout-shift", "budget": 0.1 },
        { "metric": "total-blocking-time", "budget": 300 }
      ]
    }
  ]
}
```

## Real User Monitoring (RUM) Setup

### web-vitals Library Integration

```javascript
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,  // 'good', 'needs-improvement', 'poor'
    delta: metric.delta,
    id: metric.id,
    navigationType: metric.navigationType,
    // Attribution for debugging
    entries: metric.entries?.map(e => ({
      element: e.element?.tagName,
      url: e.url,
      loadTime: e.loadTime,
      renderTime: e.renderTime,
      size: e.size,
    })),
  });

  // Use sendBeacon for reliability (works during page unload)
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/vitals', body);
  } else {
    fetch('/api/vitals', { body, method: 'POST', keepalive: true });
  }
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

### Attribution Builds (Detailed Debugging)

```javascript
import { onLCP, onINP, onCLS } from 'web-vitals/attribution';

onLCP((metric) => {
  console.log('LCP Attribution:', {
    element: metric.attribution.element,
    url: metric.attribution.url,
    timeToFirstByte: metric.attribution.timeToFirstByte,
    resourceLoadDelay: metric.attribution.resourceLoadDelay,
    resourceLoadDuration: metric.attribution.resourceLoadDuration,
    elementRenderDelay: metric.attribution.elementRenderDelay,
  });
});

onINP((metric) => {
  console.log('INP Attribution:', {
    eventTarget: metric.attribution.eventTarget,
    eventType: metric.attribution.eventType,
    inputDelay: metric.attribution.inputDelay,
    processingDuration: metric.attribution.processingDuration,
    presentationDelay: metric.attribution.presentationDelay,
    longAnimationFrameEntries: metric.attribution.longAnimationFrameEntries,
  });
});
```
