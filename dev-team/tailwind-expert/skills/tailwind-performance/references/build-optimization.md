# Build Optimization

This reference covers the full build pipeline for Tailwind CSS: PostCSS configuration, CSS minification strategies, critical CSS extraction, preloading techniques, content path optimization, production readiness, and caching strategies.

## PostCSS Pipeline

Tailwind CSS operates as a PostCSS plugin. Understanding its position in the plugin chain is essential for correct output and optimal build performance.

### Plugin Chain Order

Configure `postcss.config.js` with Tailwind first and Autoprefixer second:

```javascript
// postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

Tailwind must run before Autoprefixer because Tailwind generates the utility CSS that Autoprefixer then processes to add vendor prefixes. Placing Autoprefixer before Tailwind results in unprefixed utilities in the output.

### v4 Build Options

In Tailwind v4, the PostCSS plugin remains available but is no longer the only integration path:

- **Standalone CLI**: Run `npx @tailwindcss/cli -i input.css -o output.css` without any PostCSS configuration. Suitable for projects that do not need other PostCSS plugins.
- **Vite plugin**: Use `@tailwindcss/vite` for native Vite integration. This bypasses PostCSS entirely, leveraging Vite's plugin system for faster hot module replacement.
- **PostCSS plugin**: Continue using `@tailwindcss/postcss` when other PostCSS plugins (such as `postcss-import` or `postcss-nesting`) are required in the pipeline.

To choose the right integration, assess whether the project requires additional PostCSS transforms. If not, prefer the standalone CLI or Vite plugin for reduced configuration and faster builds.

### Plugin Chain Performance

To keep PostCSS processing fast:

- Remove unused PostCSS plugins from the chain. Each plugin adds processing time proportional to the CSS size.
- Avoid plugins that perform full AST traversals when a simpler alternative exists.
- Measure plugin impact by temporarily disabling individual plugins and comparing build times.
- In v4, prefer the Vite plugin or standalone CLI to eliminate PostCSS overhead entirely when no other PostCSS plugins are needed.

## CSS Minification

Minification removes whitespace, comments, and redundant declarations from the CSS output to reduce file size.

### cssnano

cssnano is a modular CSS minifier built on PostCSS. To add it to the pipeline:

```javascript
// postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
    ...(process.env.NODE_ENV === 'production' ? { cssnano: { preset: 'default' } } : {}),
  },
};
```

cssnano applies a suite of optimizations: merging duplicate rules, reducing `calc()` expressions, normalizing values, and stripping comments. Enable it only in production to avoid slowing down development rebuilds.

### Tailwind's Built-in Minification

Use the `--minify` flag with the Tailwind CLI for quick minification without adding cssnano to the pipeline:

```bash
npx tailwindcss -o output.css --minify
```

This applies basic whitespace and comment removal. For most projects, it produces output comparable to cssnano's default preset.

### Lightning CSS

Tailwind v4 uses Lightning CSS as its default CSS transformation and minification engine. Lightning CSS is written in Rust and performs significantly faster than JavaScript-based tools.

Benefits of Lightning CSS:

- Handles minification, vendor prefixing, and CSS nesting in a single pass.
- Eliminates the need for separate Autoprefixer and cssnano plugins when using the v4 Vite plugin or standalone CLI.
- Produces output with smaller file sizes due to more aggressive optimization passes.

### Comparison of Minification Tools

| Tool | Speed | Output Size | Integration |
|------|-------|------------|-------------|
| cssnano | Moderate | Excellent | PostCSS plugin |
| Lightning CSS | Fast (Rust) | Excellent | Built into Tailwind v4 |
| esbuild CSS | Fast (Go) | Good | Bundler integration |
| Tailwind `--minify` | Fast | Good | CLI flag |

For v3 projects, use cssnano in the PostCSS pipeline. For v4 projects, rely on Lightning CSS through the default toolchain. For projects using esbuild as the bundler, leverage its built-in CSS minification to avoid adding a separate tool.

## Critical CSS Extraction

Critical CSS refers to the minimal set of styles required to render above-the-fold content. Inlining these styles in the HTML `<head>` eliminates the render-blocking network request for the full stylesheet on initial page load.

### How Critical CSS Works

1. A tool renders the page (or simulates a viewport) and identifies which CSS rules apply to elements visible in the initial viewport.
2. Those rules are extracted and inlined as a `<style>` block in the HTML `<head>`.
3. The full stylesheet is loaded asynchronously so it does not block rendering.

### Tools for Critical CSS Extraction

**`critical` npm package**: A standalone tool that generates critical CSS for given HTML files.

```bash
npm install -D critical
```

```javascript
const critical = require('critical');

critical.generate({
  base: 'dist/',
  src: 'index.html',
  css: ['dist/styles.css'],
  inline: true,
  dimensions: [
    { width: 375, height: 667 },   // mobile
    { width: 1440, height: 900 },  // desktop
  ],
});
```

**`critters`**: A Webpack and Vite plugin that inlines critical CSS at build time without launching a headless browser. It parses the HTML and CSS statically, making it faster but slightly less accurate than browser-based tools.

```javascript
// vite.config.js
import critters from 'critters';

export default {
  plugins: [critters()],
};
```

### Critical CSS with Tailwind

Tailwind's utility classes are granular, which works well with critical CSS extraction. The tools identify which utilities are used by above-the-fold elements and inline only those declarations. The rest of the Tailwind output loads asynchronously.

To maximize the benefit:

- Keep above-the-fold components simple with a limited set of utilities.
- Avoid large `@apply` blocks in component CSS, as they increase the critical CSS payload.
- Test critical CSS output across multiple viewport sizes to ensure coverage.

The primary benefit is faster First Contentful Paint (FCP), as the browser can render styled content without waiting for the full stylesheet to download.

## Preloading CSS

Preloading tells the browser to fetch the stylesheet with high priority before it discovers the `<link>` tag during HTML parsing.

### Preload Link Tag

```html
<link rel="preload" href="/styles.[hash].css" as="style" />
<link rel="stylesheet" href="/styles.[hash].css" />
```

The `preload` hint starts the download immediately. The browser still applies the stylesheet when the `<link rel="stylesheet">` tag is encountered, but the file is already in the cache.

### HTTP/2 Server Push

Configure the server to push CSS assets alongside the HTML response:

```
Link: </styles.[hash].css>; rel=preload; as=style
```

This eliminates the round-trip delay between receiving the HTML and requesting the CSS. Note that HTTP/2 push has been deprecated in some browsers and CDN providers; verify support before relying on it. Prefer `103 Early Hints` as the modern replacement.

### Split CSS Strategy

To combine critical CSS with preloading:

1. Inline critical CSS in the HTML `<head>` as a `<style>` block.
2. Preload the full stylesheet so it downloads in parallel with HTML parsing.
3. Apply the full stylesheet asynchronously using the `media` attribute pattern:

```html
<style>/* inlined critical CSS */</style>
<link rel="preload" href="/styles.[hash].css" as="style" onload="this.rel='stylesheet'" />
<noscript><link rel="stylesheet" href="/styles.[hash].css" /></noscript>
```

This ensures styled above-the-fold content appears immediately while the full stylesheet loads without blocking rendering.

## Content Path Optimization

Overly broad content paths slow down the scanner and may detect class strings in irrelevant files, inflating the output.

### Be Specific with Globs

```javascript
// v3 tailwind.config.js
content: [
  './src/components/**/*.{js,ts,jsx,tsx}',
  './src/pages/**/*.{js,ts,jsx,tsx}',
  './src/layouts/**/*.{js,ts,jsx,tsx}',
  './public/index.html',
],
```

Avoid catching everything with `./src/**/*`. That pattern includes test files, mock data, Storybook stories, documentation, and other files that may contain class-like strings but never render in production.

### Exclude Non-Production Files

Keep test files, stories, and documentation out of content paths:

- `**/*.test.{ts,tsx}` — unit tests often contain class strings for assertions
- `**/*.stories.{ts,tsx}` — Storybook stories may reference classes not used in production
- `**/docs/**` — documentation files with example class names

If these files need Tailwind classes (such as Storybook), configure a separate Tailwind build for the development environment.

### Monorepo Considerations

In large monorepos, scope content paths to the packages that share the Tailwind configuration:

```javascript
content: [
  './packages/ui/src/**/*.{js,ts,jsx,tsx}',
  './packages/web/src/**/*.{js,ts,jsx,tsx}',
  // Do NOT include ./packages/api/** — backend code has no Tailwind classes
],
```

Scanning irrelevant packages wastes build time and risks false positives from coincidental string matches.

### Third-Party Component Libraries

When using component libraries that ship with Tailwind classes (such as Headless UI, Radix with Tailwind styling, or internal shared packages), add their installed paths to the content configuration:

```javascript
content: [
  './src/**/*.{js,ts,jsx,tsx}',
  './node_modules/@your-org/ui-library/dist/**/*.js',
],
```

Without this, classes used only inside the library are not detected and their styles are missing from the output.

## Production Build Checklist

Follow this checklist before deploying to production to ensure optimal CSS output:

1. **Content paths include all files with Tailwind classes.** Verify that every component, page, layout, and template file is covered by the `content` globs. Missing paths result in missing styles.

2. **No dynamic class interpolation in source.** Search the codebase for template literals and concatenation patterns that construct class names dynamically. Replace them with lookup objects, CVA definitions, or conditional `cn()` calls.

3. **CSS minification enabled.** Confirm that the build pipeline includes minification via cssnano, Lightning CSS, the `--minify` CLI flag, or the bundler's built-in CSS minification.

4. **Unused safelist entries removed.** Audit the safelist for classes that are no longer needed. Each safelisted class is included unconditionally and adds to the output size.

5. **Source maps disabled in production.** CSS source maps increase file size and expose implementation details. Disable them for production builds unless debugging in production is explicitly required.

6. **Gzip or Brotli compression configured on the server.** Ensure the web server or CDN compresses CSS responses. Brotli typically achieves 15-20% better compression than Gzip for CSS files.

7. **CSS budget threshold enforced in CI.** Add a CI step that measures the gzipped CSS size and fails the build if it exceeds the defined budget (e.g., 30KB gzipped). This prevents gradual, unnoticed growth.

## Caching Strategies

Proper caching ensures that users download CSS only when it changes, while always receiving the latest version after a deployment.

### Content-Hash Filenames

Configure the build tool to include a content hash in the CSS filename:

```
styles.a3f2b9c1.css
```

The hash changes only when the CSS content changes. This allows setting aggressive cache headers because the URL itself is the cache key. Frameworks like Vite and Next.js generate content-hashed filenames by default.

### Cache Headers

Set long-lived cache headers for hashed CSS assets:

```
Cache-Control: public, max-age=31536000, immutable
```

The `immutable` directive tells browsers that the file at this URL will never change, eliminating conditional revalidation requests. Apply this only to files with content hashes in their names.

### CDN Caching

Configure the CDN to cache CSS assets at edge locations:

- Set `Cache-Control` and `Surrogate-Control` headers appropriately.
- Purge the CDN cache on deployment if using non-hashed filenames (not recommended).
- With content-hashed filenames, CDN purging is unnecessary because new deployments produce new URLs.

### Service Worker Caching

For progressive web applications, cache CSS assets in a service worker for offline support:

```javascript
// service-worker.js
const CACHE_NAME = 'styles-v1';
const CSS_URLS = ['/styles.a3f2b9c1.css'];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(CSS_URLS))
  );
});

self.addEventListener('fetch', (event) => {
  if (event.request.url.endsWith('.css')) {
    event.respondWith(
      caches.match(event.request).then((response) => response || fetch(event.request))
    );
  }
});
```

Use a cache-first strategy for hashed CSS assets and a network-first strategy for non-hashed assets to balance offline support with freshness.

### Cache Invalidation

Content-hash filenames provide automatic cache invalidation. When the CSS changes, the hash changes, and the new filename is referenced in the updated HTML. Old cached files expire naturally or can be cleaned up during deployment.

To verify correct caching behavior:

- Inspect response headers in browser DevTools to confirm `Cache-Control` values.
- Test cache invalidation by making a CSS change, deploying, and verifying that the browser fetches the new file.
- Monitor cache hit rates on the CDN to confirm that assets are being served from edge locations.
