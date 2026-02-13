# Resource Loading Strategies

## Preload, Prefetch, and Preconnect

### Preload (Current Page, High Priority)

Load resources needed for the current page ASAP:

```html
<!-- Critical font file -->
<link rel="preload" as="font" type="font/woff2"
      href="/fonts/brand.woff2" crossorigin />

<!-- LCP image -->
<link rel="preload" as="image" href="/images/hero.webp"
      fetchpriority="high" type="image/webp" />

<!-- Critical CSS -->
<link rel="preload" as="style" href="/css/critical.css" />

<!-- Critical script -->
<link rel="preload" as="script" href="/js/app.js" />
```

**Rules:**
- Only preload resources needed in the first ~3 seconds.
- Excessive preloads compete for bandwidth and slow everything down.
- Match the `as` attribute to the resource type (font, image, style, script).
- Add `crossorigin` for fonts (always) and cross-origin resources.
- Use `type` for images to avoid preloading unsupported formats.

### Prefetch (Next Page, Low Priority)

Speculatively load resources for the next likely navigation:

```html
<!-- Prefetch next page HTML -->
<link rel="prefetch" href="/products/popular-item" />

<!-- Prefetch next page assets -->
<link rel="prefetch" as="script" href="/js/product-page.js" />
```

**Rules:**
- Only prefetch resources with high probability of being needed.
- Prefetch uses idle bandwidth — does not compete with current page.
- Browser may ignore prefetch hints on slow connections or limited data.

### Preconnect (Resolve DNS + TCP + TLS Early)

Warm up connections to origins that will be used soon:

```html
<!-- CDN for images -->
<link rel="preconnect" href="https://cdn.example.com" />

<!-- Google Fonts -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Analytics -->
<link rel="preconnect" href="https://www.google-analytics.com" />
```

**Rules:**
- Limit to 4-6 origins (each preconnect holds a socket open).
- Use `dns-prefetch` as fallback for older browsers:
  ```html
  <link rel="preconnect" href="https://cdn.example.com" />
  <link rel="dns-prefetch" href="https://cdn.example.com" />
  ```

### Speculation Rules API (Chrome 121+)

Modern replacement for prefetch with more control:

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "urls": ["/products/popular-item", "/about"],
      "eagerness": "moderate"
    }
  ],
  "prefetch": [
    {
      "where": {
        "href_matches": "/products/*"
      },
      "eagerness": "conservative"
    }
  ]
}
</script>
```

**Eagerness levels:**
- `immediate`: Speculate as soon as rules are observed.
- `eager`: Speculate as soon as possible (slightly after immediate).
- `moderate`: Speculate on hover (200ms delay).
- `conservative`: Speculate on mouse/pointer down.

## Script Loading Patterns

### async vs defer vs module

```html
<!-- Regular: blocks HTML parsing, fetches and executes immediately -->
<script src="app.js"></script>

<!-- async: fetches during parsing, executes ASAP (may interrupt parsing) -->
<script src="analytics.js" async></script>

<!-- defer: fetches during parsing, executes after DOM is ready, in order -->
<script src="app.js" defer></script>

<!-- module: always deferred, supports import/export -->
<script type="module" src="app.js"></script>
```

### When to Use Each

| Pattern | Use Case | Execution Order | Blocks Parser |
|---------|----------|-----------------|---------------|
| `defer` | App code that needs DOM | Guaranteed order | No |
| `async` | Analytics, ads, independent scripts | No guarantee | Yes (briefly) |
| `module` | Modern ES module code | Guaranteed order | No |
| Inline `<script>` | Critical bootstrap code | Immediate | Yes |

### Recommended Loading Strategy

```html
<head>
  <!-- Critical CSS inlined -->
  <style>/* critical above-fold styles */</style>

  <!-- Preload LCP image -->
  <link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />

  <!-- Preconnect to external origins -->
  <link rel="preconnect" href="https://cdn.example.com" />

  <!-- Main app script deferred -->
  <script src="/js/app.js" defer></script>
</head>
<body>
  <!-- Content -->

  <!-- Non-critical scripts at end, async -->
  <script src="/js/analytics.js" async></script>
</body>
```

## CSS Delivery Optimization

### Critical CSS Extraction

Extract CSS needed for above-the-fold content and inline it:

```html
<head>
  <!-- Inlined critical CSS -->
  <style>
    /* Only styles needed for initial viewport render */
    :root { --primary: #1a73e8; }
    body { margin: 0; font-family: system-ui; }
    .header { display: flex; height: 64px; }
    .hero { min-height: 400px; }
    /* ... minimal set for above-fold content */
  </style>

  <!-- Full stylesheet loaded asynchronously -->
  <link rel="preload" href="/css/styles.css" as="style"
        onload="this.onload=null;this.rel='stylesheet'" />
  <noscript><link rel="stylesheet" href="/css/styles.css" /></noscript>
</head>
```

### CSS Code Splitting

Split CSS by route or component to avoid loading unused styles:

```
styles/
├── global.css      (10KB — loaded everywhere)
├── home.css        (5KB — homepage only)
├── product.css     (8KB — product pages only)
├── checkout.css    (6KB — checkout flow only)
└── components/
    ├── header.css  (2KB — loaded everywhere)
    └── footer.css  (2KB — loaded everywhere)
```

### Removing Unused CSS

Tools for identifying and removing unused CSS:
- **PurgeCSS**: Scans HTML/JS for used selectors, removes unused.
- **Chrome DevTools Coverage tab**: Shows per-file used vs unused bytes.
- **Tailwind CSS**: Purges unused utilities by default in production.

## Image Optimization

### Format Selection

| Format | Best For | Browser Support | Compression |
|--------|----------|-----------------|-------------|
| AVIF | Photos, illustrations | Chrome, Firefox, Safari 16+ | Best (50-70% smaller than JPEG) |
| WebP | Photos, illustrations | All modern browsers | Good (25-35% smaller than JPEG) |
| JPEG | Photos (fallback) | Universal | Baseline |
| PNG | Transparency, screenshots | Universal | Lossless |
| SVG | Icons, logos, illustrations | Universal | Vector (tiny) |

### Responsive Image Pattern

```html
<picture>
  <!-- AVIF for browsers that support it -->
  <source type="image/avif"
          srcset="/img/product-400.avif 400w,
                 /img/product-800.avif 800w,
                 /img/product-1200.avif 1200w"
          sizes="(max-width: 600px) 100vw,
                 (max-width: 1200px) 50vw,
                 33vw" />

  <!-- WebP fallback -->
  <source type="image/webp"
          srcset="/img/product-400.webp 400w,
                 /img/product-800.webp 800w,
                 /img/product-1200.webp 1200w"
          sizes="(max-width: 600px) 100vw,
                 (max-width: 1200px) 50vw,
                 33vw" />

  <!-- JPEG final fallback -->
  <img src="/img/product-800.jpg"
       alt="Product description"
       width="1200" height="800"
       loading="lazy"
       decoding="async" />
</picture>
```

### Lazy Loading

```html
<!-- Below-the-fold images: lazy load -->
<img src="photo.jpg" alt="Description"
     loading="lazy" decoding="async"
     width="800" height="600" />

<!-- Above-the-fold / LCP image: eager load with high priority -->
<img src="hero.jpg" alt="Hero"
     fetchpriority="high" decoding="async"
     width="1200" height="630" />
```

**Never lazy load:**
- The LCP element
- Images visible in the initial viewport without scrolling
- Critical logo or branding images

### Image CDN Benefits

Image CDNs (Cloudinary, imgix, Cloudflare Images) provide:
- Automatic format negotiation (AVIF/WebP/JPEG based on Accept header).
- On-the-fly resizing via URL parameters.
- Global edge caching.
- Automatic quality optimization.

```html
<!-- Cloudflare Images example -->
<img src="https://imagedelivery.net/HASH/image-id/w=800,h=600,fit=cover,f=auto"
     alt="Product" width="800" height="600" />
```

## Font Loading Strategies

### Optimal Font Loading

```css
@font-face {
  font-family: 'BrandFont';
  src: url('/fonts/brand.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC;
}
```

```html
<!-- Preload critical font -->
<link rel="preload" as="font" type="font/woff2"
      href="/fonts/brand.woff2" crossorigin />
```

### font-display Values

| Value | Behavior | Use When |
|-------|----------|----------|
| `swap` | Show fallback immediately, swap when loaded | Brand fonts, body text |
| `optional` | Show fallback, swap only if loaded very fast | Nice-to-have fonts |
| `fallback` | Brief invisible period, then fallback, then swap | Compromise approach |
| `block` | Invisible text for up to 3s, then fallback | Icon fonts only |

### Reducing Font CLS

Use CSS `size-adjust`, `ascent-override`, `descent-override` to match fallback font metrics:

```css
/* System font fallback that closely matches custom font */
@font-face {
  font-family: 'BrandFont-Fallback';
  src: local('Arial');
  size-adjust: 105.2%;
  ascent-override: 92%;
  descent-override: 22%;
  line-gap-override: 0%;
}

body {
  font-family: 'BrandFont', 'BrandFont-Fallback', Arial, sans-serif;
}
```

Use the `fontaine` or `@next/font` (Next.js) tools to calculate these overrides automatically.

## Caching Strategies

### Cache-Control Headers

```
# Immutable versioned assets (JS, CSS, images with hash in filename)
Cache-Control: public, max-age=31536000, immutable

# HTML pages (always revalidate)
Cache-Control: public, max-age=0, must-revalidate

# API responses (short cache with stale-while-revalidate)
Cache-Control: public, max-age=60, stale-while-revalidate=600

# Private/authenticated content
Cache-Control: private, no-cache
```

### stale-while-revalidate Pattern

Serve cached content immediately while fetching a fresh copy in the background:

```
Cache-Control: public, max-age=300, stale-while-revalidate=86400
```

- For the first 300 seconds: serve from cache directly.
- From 300s to 86700s: serve stale cache, revalidate in background.
- After 86700s: wait for fresh response.

### Service Worker Caching

```javascript
// Cache-first strategy for static assets
self.addEventListener('fetch', (event) => {
  if (event.request.destination === 'image' ||
      event.request.destination === 'style' ||
      event.request.destination === 'script') {
    event.respondWith(
      caches.match(event.request).then(cached =>
        cached || fetch(event.request).then(response => {
          const clone = response.clone();
          caches.open('static-v1').then(cache =>
            cache.put(event.request, clone)
          );
          return response;
        })
      )
    );
  }
});
```

## HTTP/2 and HTTP/3

### HTTP/2 Benefits for Performance

- **Multiplexing**: Multiple requests over a single connection (no head-of-line blocking at HTTP level).
- **Header compression (HPACK)**: Reduces redundant header bytes.
- **Server Push** (deprecated in most browsers): Use `<link rel="preload">` instead.

**Optimization changes for HTTP/2:**
- Domain sharding is harmful — consolidate assets to fewer origins.
- Sprite sheets less necessary — individual files are fine with multiplexing.
- Concatenation less critical — but still useful for reducing request count.

### HTTP/3 (QUIC) Benefits

- **No TCP head-of-line blocking**: Packet loss affects only the specific stream, not all streams.
- **0-RTT connection resumption**: Faster repeat visits.
- **Better mobile performance**: Handles network transitions (WiFi → cellular) without reconnecting.

Enable HTTP/3 by configuring your CDN or reverse proxy (Cloudflare, AWS CloudFront, nginx with quic module).
