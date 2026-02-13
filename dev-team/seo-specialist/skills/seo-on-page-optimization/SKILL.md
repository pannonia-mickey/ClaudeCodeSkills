---
name: SEO On-Page Optimization
description: This skill should be used when the user asks to "optimize meta tags", "fix heading structure", "improve title tags", "write meta descriptions", "optimize internal links", "fix alt text", "improve URL structure", "add Open Graph tags", "add Twitter Cards", or mentions on-page SEO elements like keyword density, content structure, or anchor text.
---

## Purpose

Provide actionable patterns for optimizing on-page HTML elements that directly influence search engine ranking and click-through rates. On-page optimization ensures that each page clearly communicates its topic to both search engines and users.

## Meta Tags

### Title Tag

The `<title>` tag is the single most impactful on-page ranking factor.

```html
<!-- Pattern: Primary Keyword — Secondary Keyword | Brand -->
<title>Wireless Headphones — Noise Cancelling | AudioShop</title>
```

**Rules:**
- 50-60 characters maximum (Google truncates at ~580px display width).
- Primary keyword near the beginning.
- Unique per page — no duplicate titles across the site.
- Brand name at the end, separated by a pipe `|` or dash `—`.
- Avoid keyword stuffing or ALL CAPS.

### Meta Description

Not a ranking factor but directly affects click-through rate (CTR) from search results.

```html
<meta name="description" content="Shop premium wireless noise-cancelling headphones with 40-hour battery life. Free shipping on orders over $50. Compare top brands and read verified reviews." />
```

**Rules:**
- 150-160 characters maximum.
- Include primary keyword naturally (Google bolds matching terms).
- Actionable language with a value proposition.
- Unique per page — no duplicates.
- Avoid generic descriptions that could apply to any page.

### Open Graph and Twitter Cards

Essential for social sharing and rich link previews.

```html
<!-- Open Graph -->
<meta property="og:title" content="Wireless Headphones — Noise Cancelling" />
<meta property="og:description" content="Premium noise-cancelling headphones with 40-hour battery." />
<meta property="og:image" content="https://example.com/images/headphones-og.jpg" />
<meta property="og:url" content="https://example.com/products/headphones" />
<meta property="og:type" content="product" />
<meta property="og:site_name" content="AudioShop" />

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content="Wireless Headphones — Noise Cancelling" />
<meta name="twitter:description" content="Premium noise-cancelling headphones with 40-hour battery." />
<meta name="twitter:image" content="https://example.com/images/headphones-twitter.jpg" />
```

**OG image requirements:** Minimum 1200x630px, under 8MB, JPEG or PNG.

## Heading Hierarchy

### Rules

- Exactly one `<h1>` per page reflecting the primary topic.
- `<h2>` for major sections, `<h3>` for subsections — never skip levels.
- Headings must form a logical outline of the page content.
- Include target keywords naturally in H1 and H2 tags without stuffing.

```html
<h1>Wireless Noise-Cancelling Headphones</h1>              <!-- Primary topic -->
  <h2>Features</h2>                                          <!-- Major section -->
    <h3>Active Noise Cancellation</h3>                       <!-- Subsection -->
    <h3>40-Hour Battery Life</h3>
  <h2>Customer Reviews</h2>
    <h3>Sound Quality</h3>
    <h3>Comfort and Fit</h3>
  <h2>Compare Models</h2>
```

### Common Mistakes

- Multiple `<h1>` tags (use one per page, not one per component).
- Skipping from `<h2>` to `<h4>` without `<h3>`.
- Using headings for visual styling instead of document structure.
- Generic headings ("Overview", "Details") that waste ranking potential.

## Internal Linking

### Anchor Text

Use descriptive anchor text that tells search engines what the target page is about.

```html
<!-- Good: descriptive anchor text -->
<a href="/guides/noise-cancellation">how active noise cancellation works</a>

<!-- Bad: generic anchor text -->
<a href="/guides/noise-cancellation">click here</a>
<a href="/guides/noise-cancellation">read more</a>
<a href="/guides/noise-cancellation">learn more</a>
```

### Linking Strategy

- Link from high-authority pages to important pages that need ranking boosts.
- Use contextual links within body content (more valuable than navigation links).
- Ensure every important page is reachable within 3 clicks from the homepage.
- Fix orphan pages by adding links from relevant content or navigation.
- Avoid excessive links on a single page (dilutes link equity).

## Image Optimization

### Alt Text

```html
<!-- Good: descriptive, concise -->
<img src="headphones.jpg" alt="Black wireless over-ear headphones with memory foam ear cushions" />

<!-- Bad: keyword stuffed -->
<img src="headphones.jpg" alt="headphones wireless headphones best headphones noise cancelling headphones" />

<!-- Bad: empty or generic -->
<img src="headphones.jpg" alt="image" />
<img src="headphones.jpg" alt="" />  <!-- Only acceptable for decorative images -->
```

**Rules:**
- Describe the image content concisely (125 characters max for screen readers).
- Include relevant keywords naturally when they describe the image.
- Use empty `alt=""` only for purely decorative images (CSS backgrounds preferred).
- Always include `width` and `height` attributes to prevent CLS.

## URL Structure

### Best Practices

```
Good:  /products/wireless-headphones
Good:  /blog/how-to-choose-headphones
Bad:   /products?id=12345&cat=audio
Bad:   /p/wl-hp-nc-2024-v3
Bad:   /products/Wireless_Headphones_Best_Price_2024
```

**Rules:**
- Lowercase, hyphen-separated words.
- Short, descriptive, and human-readable.
- Include primary keyword when natural.
- Avoid query parameters for content pages (use path segments).
- Avoid dates in URLs unless content is inherently time-bound (news articles).
- Avoid unnecessary nesting depth (`/a/b/c/d/e/page` — flatten where possible).

## Content Structure Patterns

### Above the Fold

Place the primary keyword in the first 100 words. The opening paragraph should clearly state the page topic and user value proposition.

### Content Formatting for SEO

- Short paragraphs (2-4 sentences) for readability.
- Bulleted and numbered lists for scannable content (often featured in snippets).
- Bold key phrases to help both users and search engines identify important terms.
- Table of contents for long-form content (creates jump links in search results).
- FAQ sections at the bottom (eligible for FAQ rich results with structured data).

## References

- **[meta-tag-patterns.md](references/meta-tag-patterns.md)** — Complete meta tag reference including advanced tags (robots variants, referrer, theme-color), framework-specific implementations (Next.js generateMetadata, Nuxt useSeoMeta, Remix meta function), and templates for different page types (product, article, category, homepage).
- **[internal-linking-strategy.md](references/internal-linking-strategy.md)** — Advanced internal linking patterns including hub-and-spoke content architecture, topical clusters, silo structures, breadcrumb optimization, pagination linking, and audit methodology for identifying linking gaps.
