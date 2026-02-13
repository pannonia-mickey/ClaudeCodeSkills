---
name: SEO Technical Audit
description: This skill should be used when the user asks to "audit SEO", "check crawlability", "fix robots.txt", "generate sitemap", "check canonical URLs", "find redirect chains", "fix indexing issues", "check mobile SEO", or mentions technical SEO problems like duplicate content, crawl budget, or index bloat.
---

## Purpose

Provide a systematic framework for auditing and resolving technical SEO issues that affect search engine crawling, indexing, and ranking. Technical SEO forms the foundation — without it, on-page optimization and structured data cannot deliver results.

## Audit Process

### 1. Crawlability Assessment

Verify that search engines can discover and access all valuable pages:

- **robots.txt**: Check for overly restrictive directives that block CSS, JS, images, or entire sections. Ensure `User-agent: *` allows access to essential resources.
- **Meta robots tags**: Scan templates for `noindex`, `nofollow`, `none` directives. Flag pages unintentionally excluded from indexing.
- **X-Robots-Tag headers**: Check server responses for HTTP-level robot directives that override meta tags.
- **JavaScript rendering**: Identify content that requires JavaScript execution. Verify SSR/SSG is in place for critical content.

```
# robots.txt — balanced production example
User-agent: *
Allow: /
Disallow: /api/
Disallow: /admin/
Disallow: /cart/
Disallow: /*?sort=
Disallow: /*?filter=

Sitemap: https://example.com/sitemap.xml
```

### 2. Indexing Verification

Ensure the correct pages are indexed and duplicates are consolidated:

- **Canonical URLs**: Every indexable page must have a `<link rel="canonical">` pointing to itself or the preferred version. Paginated pages self-canonicalize (do not point page 2+ to page 1).
- **Duplicate content**: Identify URL variations (trailing slashes, www vs non-www, HTTP vs HTTPS, query parameters) that create duplicates. Consolidate with canonicals and 301 redirects.
- **Index bloat**: Find low-value pages (tag archives, empty categories, search results pages) polluting the index. Apply `noindex` or remove from sitemap.

```html
<!-- Self-referencing canonical -->
<link rel="canonical" href="https://example.com/products/widget" />

<!-- Pagination — each page self-canonicalizes -->
<link rel="canonical" href="https://example.com/blog?page=3" />
```

### 3. XML Sitemap Validation

Inspect sitemap structure and content:

- Include only indexable, canonical URLs (200 status, not noindexed, not redirected).
- Use accurate `<lastmod>` dates reflecting actual content changes, not build timestamps.
- Keep individual sitemaps under 50,000 URLs / 50MB uncompressed.
- Use a sitemap index for large sites.
- Submit sitemap in robots.txt and Google Search Console.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/products/widget</loc>
    <lastmod>2025-03-15</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```

### 4. Redirect Audit

Trace redirect chains and validate implementation:

- Flag redirect chains longer than 2 hops (each hop loses crawl efficiency).
- Ensure 301 (permanent) for moved content, 302 (temporary) only for genuinely temporary moves.
- Check for redirect loops.
- Verify HTTPS redirects: HTTP -> HTTPS should be a single 301, not HTTP -> HTTPS -> www or similar chains.
- Confirm old URLs in sitemaps and internal links are updated to final destinations.

### 5. Mobile-Friendliness

Validate mobile-first indexing readiness:

- Responsive viewport meta tag: `<meta name="viewport" content="width=device-width, initial-scale=1">`.
- Same content served to mobile and desktop (no cloaking or content parity issues).
- Tap targets at least 48x48 CSS pixels with adequate spacing.
- No horizontal scrolling at mobile widths.
- Text readable without zooming (minimum 16px base font size).

### 6. HTTP and Security

Check HTTP-level SEO factors:

- HTTPS enforced across all pages with valid SSL certificate.
- HSTS header configured for secure browsing signals.
- Proper HTTP status codes: 200 for valid pages, 301 for permanent moves, 404 for genuinely missing content (not soft 404s).
- Server response time under 200ms TTFB for crawl efficiency.

## Audit Report Format

Structure findings by priority:

```
## Critical (Blocks indexing or ranking)
- [Finding]: [Impact] → [Fix]

## Warning (Degrades SEO performance)
- [Finding]: [Impact] → [Fix]

## Info (Improvement opportunity)
- [Finding]: [Impact] → [Fix]
```

## Common Framework Patterns

### Next.js App Router
- Use `generateMetadata()` for dynamic meta tags.
- Create `app/sitemap.ts` for dynamic sitemap generation.
- Place `robots.ts` in the app directory root.
- Use `generateStaticParams()` for static page generation.

### Nuxt 3
- Use `useHead()` or `useSeoMeta()` composables for meta tags.
- Configure sitemap via `@nuxtjs/sitemap` module.
- Use `nuxt.config.ts` `routeRules` for redirect and caching configuration.

### Astro
- Set meta tags in frontmatter or layout components.
- Use `@astrojs/sitemap` integration.
- Static output by default ensures full crawlability.

## References

- **[crawlability-checklist.md](references/crawlability-checklist.md)** — Detailed crawlability checklist covering robots.txt patterns, meta robots combinations, URL parameter handling, faceted navigation strategies, crawl budget optimization, and international SEO with hreflang.
- **[indexing-troubleshooting.md](references/indexing-troubleshooting.md)** — Diagnostic guide for common indexing problems including soft 404s, orphan pages, thin content, index bloat, canonicalization conflicts, and Google Search Console coverage report interpretation.
