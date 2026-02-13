# Crawlability Checklist

## robots.txt Patterns

### Allow/Disallow Priority

Directives are matched by specificity (longest matching path wins), not order:

```
User-agent: *
Allow: /products/
Disallow: /products/internal/
# /products/widget → Allowed
# /products/internal/api → Disallowed
```

### Per-Bot Directives

```
User-agent: Googlebot
Allow: /

User-agent: Bingbot
Allow: /

User-agent: GPTBot
Disallow: /

User-agent: *
Disallow: /admin/
```

Specific user-agent blocks override the wildcard `*` block entirely. If Googlebot has its own block, it ignores the `*` rules.

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `Disallow: /css/` | Blocks stylesheets needed for rendering | Remove — bots need CSS |
| `Disallow: /js/` | Blocks JavaScript needed for rendering | Remove — bots need JS for SPA content |
| `Disallow: /images/` | Blocks image indexing in Google Images | Remove unless images are private |
| No `Sitemap:` directive | Bots must discover sitemap elsewhere | Add `Sitemap: https://example.com/sitemap.xml` |
| `Disallow: /` in staging | Staging robots.txt deployed to production | Use environment-specific robots.txt |

### Wildcards

```
# Block all query string URLs
Disallow: /*?

# Block specific parameter combinations
Disallow: /*?sort=
Disallow: /*?filter=
Disallow: /*?page=*&sort=

# Allow specific query string URLs
Allow: /*?page=
```

## Meta Robots Combinations

| Directive | Meaning |
|-----------|---------|
| `index, follow` | Default behavior — index page, follow links (no tag needed) |
| `noindex, follow` | Do not index this page, but follow its links to discover other pages |
| `index, nofollow` | Index this page, but do not follow its outbound links |
| `noindex, nofollow` | Do not index, do not follow links (equivalent to `none`) |
| `noarchive` | Do not show cached copy in search results |
| `nosnippet` | Do not show text snippet or video preview in results |
| `max-snippet:50` | Limit text snippet to 50 characters |
| `max-image-preview:large` | Allow large image preview in results |
| `max-video-preview:30` | Allow up to 30 seconds of video preview |

### X-Robots-Tag HTTP Header

Apply robot directives at the HTTP level (useful for PDFs, images, non-HTML resources):

```
X-Robots-Tag: noindex, nofollow
X-Robots-Tag: googlebot: noindex
X-Robots-Tag: unavailable_after: 2025-12-25T00:00:00+00:00
```

## URL Parameter Handling

### Faceted Navigation Strategy

Faceted navigation (filters, sorts) creates exponential URL combinations that waste crawl budget:

**Problem:**
```
/products?color=red
/products?color=red&size=large
/products?color=red&size=large&sort=price
/products?size=large&color=red  (duplicate with different param order)
```

**Solutions:**

1. **Canonical to base page**: All filtered views canonicalize to the unfiltered category page.
   - Use when filters do not create meaningfully different content.
   ```html
   <!-- On /products?color=red&size=large -->
   <link rel="canonical" href="https://example.com/products" />
   ```

2. **Consistent parameter ordering**: Normalize parameter order server-side to prevent duplicates.
   ```
   /products?color=red&size=large  → canonical
   /products?size=large&color=red  → 301 redirect to canonical order
   ```

3. **robots.txt blocking**: Block crawling of parameter combinations.
   ```
   Disallow: /*?sort=
   Disallow: /*?filter=
   Disallow: /*?*&*&*  # Block URLs with 3+ parameters
   ```

4. **Noindex with follow**: Allow link discovery but prevent indexing.
   ```html
   <meta name="robots" content="noindex, follow" />
   ```

5. **Static URLs for valuable filters**: If "red widgets" has search demand, create `/products/red-widgets` as a static URL with unique content.

### Sort and View Parameters

Always block from indexing — sorted views are duplicate content:

```
Disallow: /*?sort=
Disallow: /*?view=
Disallow: /*?order=
```

## Crawl Budget Optimization

Crawl budget matters primarily for large sites (100K+ pages). Optimization strategies:

1. **Remove low-value pages from crawl paths**: noindex thin content, block faceted URLs.
2. **Fix crawl traps**: Infinite calendar pages, session ID URLs, printer-friendly duplicates.
3. **Improve server response time**: Faster responses = more pages crawled per session.
4. **Update sitemaps**: Include only canonical, indexable pages. Remove 404s and redirects.
5. **Fix redirect chains**: Direct all internal links to final URLs.
6. **Reduce duplicate content**: Canonicalize or consolidate.

## International SEO — hreflang

### Implementation

```html
<!-- On English page -->
<link rel="alternate" hreflang="en" href="https://example.com/products" />
<link rel="alternate" hreflang="fr" href="https://example.com/fr/produits" />
<link rel="alternate" hreflang="de" href="https://example.com/de/produkte" />
<link rel="alternate" hreflang="x-default" href="https://example.com/products" />
```

### Rules

- **Bidirectional**: If page A references page B, page B must reference page A.
- **Self-referencing**: Every page must include a hreflang pointing to itself.
- **x-default**: Always include for the fallback/default language version.
- **Canonical consistency**: The canonical URL must match the hreflang URL for that language.
- **ISO codes**: Use ISO 639-1 for language, optionally ISO 3166-1 Alpha 2 for region (`en-US`, `en-GB`).

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Missing return links | Page A → B but B ↛ A | Add bidirectional references |
| Missing self-reference | Page does not reference itself | Add hreflang for own URL |
| No x-default | No fallback for unlisted languages | Add x-default to every set |
| Canonical mismatch | Canonical is `/en/` but hreflang points to `/en/?ref=nav` | Align canonical and hreflang URLs exactly |
| Wrong ISO codes | `hreflang="english"` | Use `hreflang="en"` |

## Mobile SEO Audit Points

### Viewport Configuration

```html
<!-- Required for mobile-first indexing -->
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

### Content Parity

Google indexes the mobile version of pages. Ensure:

- Same text content on mobile and desktop.
- Same structured data on both versions.
- Same meta tags (title, description, robots) on both.
- Same internal links accessible on mobile.
- Images have same alt text on both versions.

### Mobile Usability Issues

- **Text too small**: Base font size below 16px.
- **Clickable elements too close**: Tap targets smaller than 48x48px or less than 8px apart.
- **Content wider than screen**: Horizontal scroll on mobile viewport.
- **Interstitials**: Full-screen popups that block content on mobile (Google penalty).
