# Rich Results Eligibility Guide

## Rich Result Types by Schema

| Schema Type | Rich Result Feature | Search Appearance |
|-------------|-------------------|-------------------|
| Product | Product snippet | Price, availability, rating stars, review count |
| Article | Article rich result | Headline, thumbnail, date, author |
| FAQPage | FAQ accordion | Expandable Q&A directly in results |
| HowTo | How-to steps | Step-by-step with images in results |
| Recipe | Recipe card | Image, rating, cook time, calories |
| Event | Event listing | Date, location, ticket price |
| BreadcrumbList | Breadcrumb trail | Structured path replaces URL |
| VideoObject | Video thumbnail | Thumbnail, duration, key moments |
| LocalBusiness | Local panel | Address, hours, phone, reviews |
| Organization | Knowledge panel | Logo, social links, description |
| WebSite + SearchAction | Sitelinks search | Search box in sitelinks |
| Review / AggregateRating | Review stars | Star rating in snippet |
| SoftwareApplication | App info | Rating, price, platform |
| Course | Course listing | Provider, description, price |
| JobPosting | Job listing | Title, company, location, salary |

## Eligibility Requirements

### General Requirements (All Types)

1. **Valid JSON-LD syntax**: Passes JSON parser without errors.
2. **Correct @type**: Uses exact Schema.org type name (case-sensitive).
3. **Required properties present**: Each type has mandatory fields.
4. **Content match**: Structured data matches visible page content.
5. **Accessible page**: Page is indexable (not noindexed, not blocked by robots.txt).
6. **Google policies**: Content does not violate Google's structured data guidelines.

### Per-Type Eligibility

#### Product Rich Results

**Required:**
- `name`
- `image` (at least one)
- One of: `review`, `aggregateRating`, or `offers`

**For full snippet (price + availability + rating):**
- `offers.price` + `offers.priceCurrency` + `offers.availability`
- `aggregateRating.ratingValue` + `aggregateRating.ratingCount`

**Content policy:**
- Must be a product for sale, not a category page.
- Reviews must be from actual customers.
- Price must match displayed price.

#### FAQ Rich Results

**Required:**
- `mainEntity` array with at least one Question + Answer.

**Content policy:**
- Questions and answers must be visible on the page.
- Not for user-generated Q&A (use QAPage instead).
- Not for single FAQ that serves as customer support.
- Google may limit FAQ display to 2 questions per result.

#### Article Rich Results

**Required:**
- `headline`
- `image` (at least one, minimum 696px wide)
- `author` (with name)
- `publisher` (with name and logo)
- `datePublished`

**Content policy:**
- Must be a news article, blog post, or sports article.
- Not for product descriptions or marketing pages.

## Content Policies That Disqualify Markup

### Policy Violations

| Violation | Description | Consequence |
|-----------|-------------|-------------|
| **Content mismatch** | Schema data does not match page content | Rich result removed |
| **Invisible content** | Structured data for content hidden from users | Manual action |
| **Misleading markup** | Schema type does not match actual content type | Rich result removed |
| **Self-serving reviews** | Reviews written by the business about itself | Review stars removed |
| **Fake aggregation** | AggregateRating fabricated without real reviews | Review stars removed |
| **Spammy practices** | Auto-generated or irrelevant structured data | Manual action |

### Manual Actions

Google may issue manual actions for structured data abuse:

1. **Spammy structured markup**: Markup is irrelevant to the content.
2. **Misleading content**: Structured data deceives users.

**Recovery:**
- Fix the violations across all affected pages.
- Submit a reconsideration request in Google Search Console.
- Wait for Google to review (typically 1-4 weeks).

## Testing and Monitoring Workflow

### Development Testing

1. **Write JSON-LD** and validate JSON syntax.
2. **Schema.org Validator** (validator.schema.org): Checks vocabulary correctness.
3. **Google Rich Results Test** (search.google.com/test/rich-results): Checks Google eligibility.
4. **Deploy to staging** and test with the Rich Results URL test.

### Production Monitoring

1. **Google Search Console → Enhancements**: Shows per-type status.
   - Valid items: Currently producing rich results.
   - Items with warnings: May lose rich results if not fixed.
   - Invalid items: Not eligible — errors must be fixed.

2. **Performance report with search appearance filter**:
   - Filter by "Rich results" to see CTR impact.
   - Compare rich result impressions vs standard snippets.

3. **URL Inspection tool**: Check individual page structured data status.

### Monitoring Cadence

| Check | Frequency | Tool |
|-------|-----------|------|
| Rich Results Test on new pages | Before deployment | Rich Results Test |
| Search Console Enhancements | Weekly | Google Search Console |
| Performance by search appearance | Monthly | Google Search Console |
| Full site structured data audit | Quarterly | Crawl + Rich Results Test |

## Troubleshooting: Valid Schema, No Rich Results

Structured data that validates but does not produce rich results.

### Common Causes

1. **Page not indexed**: Check with URL Inspection tool. Structured data is irrelevant if the page is not in the index.

2. **Domain too new / low authority**: Google may not show rich results for newer or lower-authority sites. Build authority over time.

3. **Google policy change**: Google periodically changes which types show rich results. FAQ rich results, for example, were significantly limited in 2023.

4. **Competition**: Rich results are not guaranteed even with valid markup. Google chooses which results to enhance.

5. **Content quality**: Low-quality content may have valid markup but not receive rich results. Improve the page content.

6. **Missing recommended properties**: While not required for validation, recommended properties increase the chance of rich results. Add all recommended fields.

7. **Image issues**: Images too small, broken, or not accessible to Googlebot. Ensure images meet minimum size requirements and are crawlable.

8. **Recent deployment**: Rich results can take days to weeks to appear after deploying structured data. Allow time for Google to recrawl and reprocess.

### Diagnostic Steps

```
1. URL Inspection → Is the page indexed?
   ├── No → Fix indexing first (crawlability, canonical, sitemap)
   └── Yes → Continue

2. Rich Results Test → Does markup validate?
   ├── Errors → Fix validation errors
   └── Valid → Continue

3. Search Console Enhancements → Any warnings?
   ├── Warnings → Fix warnings
   └── No warnings → Continue

4. Check recommended fields → All present?
   ├── Missing → Add recommended fields
   └── All present → Continue

5. Check content quality → Page provides real value?
   ├── Thin content → Improve content
   └── Quality content → Wait and monitor (may take weeks)
```

## Rich Results Performance Benchmarks

### Expected CTR Impact

| Rich Result Type | Typical CTR Uplift |
|------------------|--------------------|
| Product (price + stars) | +15-30% |
| FAQ accordion | +10-25% |
| Recipe card | +20-40% |
| Video thumbnail | +15-35% |
| Review stars (non-product) | +10-20% |
| Breadcrumbs | +5-10% |
| Event listing | +15-25% |

These are approximate ranges based on industry studies. Actual impact varies by niche, competition, and position.

### Measurement

Track rich result impact by comparing:
- CTR before and after structured data deployment (same position).
- CTR of pages with rich results vs similar pages without.
- Total clicks from "Rich results" search appearance in Search Console.
