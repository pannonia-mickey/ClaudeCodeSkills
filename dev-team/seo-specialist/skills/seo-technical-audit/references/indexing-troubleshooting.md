# Indexing Troubleshooting Guide

## Google Search Console Coverage Report

### Status Categories

| Status | Meaning | Action |
|--------|---------|--------|
| **Valid** | Page is indexed | Monitor for changes |
| **Valid with warnings** | Indexed but has issues | Fix warnings to maintain indexing |
| **Excluded** | Not indexed (intentionally or not) | Review reason — may need fixing |
| **Error** | Cannot be indexed due to errors | Fix immediately |

### Common Exclusion Reasons

#### "Discovered — currently not indexed"

Google found the URL but chose not to index it.

**Causes:**
- Page has thin or duplicate content.
- Site has low overall authority and Google is selective.
- Page is deep in the site architecture (>4 clicks from homepage).
- Crawl budget exhausted before reaching this page.

**Fixes:**
- Improve content quality and uniqueness.
- Add internal links from high-authority pages.
- Include in XML sitemap with accurate lastmod.
- Reduce competing pages with similar content (consolidate).

#### "Crawled — currently not indexed"

Google crawled the page but decided not to index it.

**Causes:**
- Content is too thin or similar to already-indexed pages.
- Page quality does not meet indexing threshold.
- Content is outdated or low-value.

**Fixes:**
- Expand content with unique, valuable information.
- Differentiate from similar pages on the site.
- Add structured data to signal content type and value.
- Improve E-E-A-T signals (Experience, Expertise, Authoritativeness, Trustworthiness).

#### "Duplicate without user-selected canonical"

Google found duplicate content and chose a canonical on its own.

**Causes:**
- Missing canonical tags.
- URL parameters creating duplicate URLs.
- WWW/non-WWW or HTTP/HTTPS variations not redirected.
- Trailing slash inconsistency.

**Fixes:**
- Add explicit `<link rel="canonical">` on every page.
- 301 redirect duplicate URL patterns to canonical versions.
- Enforce consistent URL format (trailing slash or not, www or not).

#### "Duplicate, Google chose different canonical than user"

Google disagrees with the canonical tag.

**Causes:**
- Canonical points to a page with significantly different content.
- Canonical chain leads to a redirect or 404.
- Canonical page itself is noindexed.
- Multiple signals conflict (canonical says A, redirects point to B).

**Fixes:**
- Ensure canonical target has matching content.
- Verify canonical URL returns 200 and is indexable.
- Align all signals: canonical, hreflang, sitemap, internal links.

## Soft 404 Diagnosis

A soft 404 returns HTTP 200 but displays "not found" or empty content.

### Detection

```bash
# Check HTTP status of URLs that should be 404
curl -o /dev/null -s -w "%{http_code}" https://example.com/nonexistent-page
# If this returns 200, it's a soft 404
```

### Common Causes

- Custom 404 page template returns HTTP 200 instead of 404.
- Empty category pages with zero products.
- Search results pages with no matches.
- Profile pages for deleted users.

### Fixes

- Return proper HTTP 404 status code for genuinely missing content.
- For empty category/search pages, either return 404 or add `noindex`.
- Configure the web framework to return correct status codes:

```python
# Django
from django.http import Http404
raise Http404("Page not found")

# Express.js
res.status(404).render('404');
```

```typescript
// Next.js App Router
import { notFound } from 'next/navigation';

export default async function Page({ params }) {
  const product = await getProduct(params.slug);
  if (!product) notFound(); // Returns 404
  return <ProductPage product={product} />;
}
```

## Orphan Pages

Pages not linked from any internal navigation, sitemap, or other pages.

### Discovery Methods

1. **Compare sitemap URLs with crawled URLs**: URLs in sitemap but not found during crawl are potential orphans.
2. **Analytics cross-reference**: Pages with organic traffic but no internal links pointing to them.
3. **Server log analysis**: URLs receiving Googlebot visits but absent from sitemap and internal links.

### Fixes

- Add contextual internal links from relevant pages.
- Include in navigation or footer if appropriate.
- Add to XML sitemap as a minimum baseline.
- If the page has no value, consider removing or noindexing it.

## Thin Content

Pages with insufficient content to provide value or differentiate from other pages.

### Indicators

- Fewer than 200 words of unique body text.
- Most content is boilerplate (header, footer, sidebar).
- Content is auto-generated without editorial value.
- Page exists only for internal linking or navigation purposes.

### Strategies

| Situation | Action |
|-----------|--------|
| Category page with only product grid | Add introductory content, buying guide, FAQ |
| Tag page with 1-2 posts | Consolidate tags, noindex low-count tags |
| Auto-generated location pages | Add unique local content, reviews, directions |
| Empty state pages | Noindex until content is available |
| Paginated pages (page 2, 3...) | Self-canonicalize each page, do not noindex |

## Index Bloat

Too many low-value pages in the index diluting the site's overall quality signals.

### Audit Process

1. `site:example.com` in Google — check total indexed count.
2. Compare indexed count vs intended indexable pages.
3. If indexed > intended: identify the excess pages.
4. Common sources of bloat:
   - Faceted navigation URLs
   - Internal search result pages
   - Tag/category archives with minimal content
   - Calendar or date-based archives
   - Printer-friendly page versions
   - Session ID or tracking parameter URLs

### Cleanup Strategy

1. **Noindex**: Apply `<meta name="robots" content="noindex, follow">` to low-value pages.
2. **Block crawling**: Add patterns to robots.txt (prevents future crawling, does not remove already-indexed pages).
3. **Canonical consolidation**: Point duplicates to a single preferred version.
4. **URL removal tool**: For urgent removal from search results (temporary, use alongside noindex).
5. **Monitor**: Check Search Console coverage report weekly during cleanup.

## Canonicalization Conflict Resolution

When multiple signals disagree, Google uses this approximate priority:

1. **301/308 redirect** (strongest signal)
2. **rel=canonical tag** (strong signal)
3. **XML sitemap inclusion** (moderate signal)
4. **Internal link patterns** (moderate signal)
5. **HTTPS over HTTP** (tie-breaker)
6. **Shorter URL** (tie-breaker)

### Resolution Checklist

- All signals point to the same canonical URL.
- Canonical URL returns HTTP 200.
- Canonical URL is not itself redirected.
- Canonical URL is included in the sitemap.
- Internal links use the canonical URL (not redirected variants).
- Canonical URL matches the hreflang self-reference.
