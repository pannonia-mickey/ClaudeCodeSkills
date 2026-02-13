# Internal Linking Strategy

## Link Architecture Models

### Hub and Spoke (Topic Clusters)

A central "pillar" page links to and from multiple "cluster" pages covering subtopics.

```
                    [Headphones Guide]  (Hub/Pillar)
                    /       |        \
                   /        |         \
    [Noise Cancelling] [Wireless] [Over-Ear vs In-Ear]  (Spokes/Cluster)
         |                |               |
    [Best NC 2025]   [BT Codecs]    [Comfort Guide]    (Supporting)
```

**Implementation:**
- Hub page provides comprehensive overview and links to each spoke.
- Each spoke links back to the hub and to related spokes.
- Supporting pages link up to their spoke and to the hub.
- Anchor text uses topic-relevant keywords, not generic text.

**Benefits:**
- Establishes topical authority around a subject cluster.
- Distributes page authority efficiently.
- Creates clear content hierarchy for search engines.

### Silo Structure

Content organized into strict topical silos where cross-silo linking is minimized.

```
[Homepage]
    ├── [Headphones Silo]
    │   ├── /headphones/
    │   ├── /headphones/wireless/
    │   ├── /headphones/noise-cancelling/
    │   └── /headphones/gaming/
    └── [Speakers Silo]
        ├── /speakers/
        ├── /speakers/bluetooth/
        ├── /speakers/home-theater/
        └── /speakers/portable/
```

**Rules:**
- Pages within a silo link freely to each other.
- Cross-silo links are used sparingly and only when contextually relevant.
- URL structure reflects the silo hierarchy.
- Navigation reinforces silo boundaries.

### Flat Architecture

All important pages are within 2-3 clicks of the homepage. Suitable for smaller sites.

```
[Homepage] → [Category] → [Product/Article]
    Max 3 levels deep
```

## Link Equity Distribution

### PageRank Flow Principles

- Internal links pass authority (PageRank) from one page to another.
- The more internal links a page receives, the more authority it accumulates.
- Authority is divided among all links on a page (diminishing returns with more links).
- Links higher in the page content carry slightly more weight.
- Contextual links within body content are more valuable than navigation links.

### Strategic Linking Patterns

**Boost important pages:**
```
High-authority page (homepage, popular blog post)
    └── Links to → Target page that needs ranking improvement
```

**Reclaim orphan authority:**
```
Orphan page with backlinks but no internal links pointing to it
    └── Add internal links from → Related content pages
```

**Reduce click depth:**
```
Before: Homepage → Category → Subcategory → Product (4 clicks)
After:  Homepage → Featured Product (2 clicks)
        Homepage → Category → Product (3 clicks)
```

## Breadcrumb Navigation

### SEO Implementation

```html
<nav aria-label="Breadcrumb">
  <ol>
    <li><a href="/">Home</a></li>
    <li><a href="/products">Products</a></li>
    <li><a href="/products/headphones">Headphones</a></li>
    <li aria-current="page">Wireless NC Headphones</li>
  </ol>
</nav>
```

**Rules:**
- Use `<nav>` with `aria-label="Breadcrumb"` for accessibility.
- Use `<ol>` (ordered list) to indicate hierarchy.
- Current page is the last item, not linked, with `aria-current="page"`.
- Pair with BreadcrumbList JSON-LD structured data.
- Match breadcrumb text to page titles/H1s for consistency.

## Pagination Linking

### SEO-Correct Pagination

```html
<!-- Page 1 -->
<link rel="canonical" href="https://example.com/blog" />

<!-- Page 2 -->
<link rel="canonical" href="https://example.com/blog?page=2" />

<!-- Page 3 -->
<link rel="canonical" href="https://example.com/blog?page=3" />
```

**Critical rules:**
- Each paginated page self-canonicalizes (do NOT point all pages to page 1).
- Google deprecated `rel="next"` and `rel="prev"` — they are ignored.
- Each page should have unique content visible to crawlers.
- Include paginated pages in the XML sitemap.
- Use descriptive URLs: `/blog?page=2` or `/blog/page/2`.

### Infinite Scroll SEO Fix

Infinite scroll hides content from crawlers. Provide a paginated HTML fallback:

```html
<!-- Progressive enhancement approach -->
<div id="product-list">
  <!-- Products rendered server-side -->
</div>

<!-- Paginated navigation for crawlers (hidden with JS for users) -->
<nav class="pagination" aria-label="Product pages">
  <a href="/products?page=1">1</a>
  <a href="/products?page=2">2</a>
  <a href="/products?page=3">3</a>
</nav>
```

Ensure each paginated URL returns full page HTML when accessed directly.

## Audit Methodology

### Finding Linking Gaps

1. **Crawl the site** with a tool like Screaming Frog or a custom crawler.
2. **Export internal links**: Source URL → Target URL → Anchor Text.
3. **Identify orphan pages**: Pages in the sitemap or CMS that receive zero internal links.
4. **Calculate link depth**: Pages requiring more than 3 clicks from the homepage need shortcut links.
5. **Check anchor text distribution**: Look for over-reliance on generic anchors ("click here", "learn more").
6. **Find broken internal links**: 404 responses from internal link targets.

### Anchor Text Audit

| Pattern | Assessment | Action |
|---------|-----------|--------|
| "click here" | Poor — no keyword signal | Rewrite with descriptive text |
| "read more" | Poor — generic | Replace with topic-specific text |
| Exact match keyword every time | Risky — looks manipulative | Vary anchor text naturally |
| URL as anchor text | Suboptimal | Replace with descriptive text |
| Image link without alt text | No anchor signal | Add descriptive alt to the image |
| Descriptive, varied text | Ideal | Maintain pattern |

### Link Distribution Analysis

For each important page, verify:

- Receives links from at least 3-5 other internal pages.
- Has contextual links (within body content), not just navigation.
- Anchor text varies but includes target keyword naturally.
- Is within 3 clicks of the homepage.
- Links out to related pages (not a dead end).

## Footer and Navigation Links

### Do

- Include key category/section links in the main navigation.
- Use breadcrumbs on all pages below the homepage.
- Add contextual "related products/articles" links on content pages.
- Use footer for important secondary pages (About, Contact, Terms, Privacy).

### Don't

- Stuff hundreds of links in the footer (link equity dilution).
- Use identical anchor text for all links to the same page.
- Hide links in CSS carousels or tabs that require interaction to reveal.
- Use JavaScript-only links (`onclick` navigation without `<a href>`).
- Add `nofollow` to internal links (wastes PageRank — it evaporates rather than redistributing).
