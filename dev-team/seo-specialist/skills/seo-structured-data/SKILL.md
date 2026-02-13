---
name: SEO Structured Data
description: This skill should be used when the user asks to "add structured data", "implement JSON-LD", "add schema markup", "create rich snippets", "add FAQ schema", "add product schema", "add breadcrumb schema", "add article schema", "add organization schema", or mentions Schema.org, rich results, or Google Rich Results Test.
---

## Purpose

Provide patterns for implementing Schema.org structured data using JSON-LD format to qualify pages for rich results in Google, Bing, and other search engines. Correct structured data increases search visibility through enhanced listings (star ratings, prices, FAQs, how-to steps).

## JSON-LD Fundamentals

### Format

Always use JSON-LD (JavaScript Object Notation for Linked Data) over Microdata or RDFa. Google explicitly recommends JSON-LD.

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Example Product"
}
</script>
```

**Placement:** In the `<head>` or at the end of `<body>`. JSON-LD does not need to be adjacent to the content it describes.

**Multiple schemas:** Use separate `<script>` blocks or a `@graph` array for multiple entities on one page.

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    { "@type": "BreadcrumbList", ... },
    { "@type": "Product", ... }
  ]
}
</script>
```

## Common Schema Types

### Product (E-commerce)

Eligible for: price, availability, rating, and review count in search results.

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Wireless Noise-Cancelling Headphones",
  "image": [
    "https://example.com/photos/headphones-front.jpg",
    "https://example.com/photos/headphones-side.jpg"
  ],
  "description": "Premium over-ear headphones with active noise cancellation and 40-hour battery.",
  "brand": {
    "@type": "Brand",
    "name": "AudioBrand"
  },
  "sku": "WH-1000XM5",
  "gtin13": "1234567890123",
  "offers": {
    "@type": "Offer",
    "url": "https://example.com/products/headphones",
    "priceCurrency": "USD",
    "price": "299.99",
    "priceValidUntil": "2026-12-31",
    "availability": "https://schema.org/InStock",
    "itemCondition": "https://schema.org/NewCondition",
    "seller": {
      "@type": "Organization",
      "name": "AudioShop"
    }
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.7",
    "bestRating": "5",
    "ratingCount": "1423"
  }
}
```

**Required fields:** name, image, offers (price + currency + availability).

### Article / BlogPosting

Eligible for: article rich results, Top Stories carousel.

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "How to Choose Noise-Cancelling Headphones in 2025",
  "image": "https://example.com/images/article-hero.jpg",
  "author": {
    "@type": "Person",
    "name": "Jane Smith",
    "url": "https://example.com/authors/jane-smith"
  },
  "publisher": {
    "@type": "Organization",
    "name": "AudioShop Blog",
    "logo": {
      "@type": "ImageObject",
      "url": "https://example.com/logo.png"
    }
  },
  "datePublished": "2025-03-15T08:00:00+00:00",
  "dateModified": "2025-04-01T10:30:00+00:00",
  "description": "A comprehensive guide to selecting the right noise-cancelling headphones."
}
```

**Required fields:** headline, image, author, publisher, datePublished.

### FAQ Page

Eligible for: expandable FAQ accordion directly in search results.

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "How long does the battery last?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "The battery lasts up to 40 hours with noise cancellation enabled, and up to 60 hours with it disabled."
      }
    },
    {
      "@type": "Question",
      "name": "Is the headset compatible with Bluetooth 5.3?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, it supports Bluetooth 5.3 with multipoint connection for two devices simultaneously."
      }
    }
  ]
}
```

**Rules:** Questions and answers must be visible on the page. Do not use FAQPage for user-generated Q&A (use QAPage instead).

### BreadcrumbList

Eligible for: breadcrumb trail replacing URL in search results.

```json
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    {
      "@type": "ListItem",
      "position": 1,
      "name": "Home",
      "item": "https://example.com/"
    },
    {
      "@type": "ListItem",
      "position": 2,
      "name": "Products",
      "item": "https://example.com/products"
    },
    {
      "@type": "ListItem",
      "position": 3,
      "name": "Wireless Headphones"
    }
  ]
}
```

**Note:** The last item (current page) omits the `item` URL property.

### Organization / LocalBusiness

Eligible for: knowledge panel, logo in search results.

```json
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "AudioShop",
  "url": "https://example.com",
  "logo": "https://example.com/logo.png",
  "sameAs": [
    "https://twitter.com/audioshop",
    "https://www.facebook.com/audioshop",
    "https://www.linkedin.com/company/audioshop"
  ],
  "contactPoint": {
    "@type": "ContactPoint",
    "telephone": "+1-800-555-0199",
    "contactType": "customer service",
    "availableLanguage": ["English"]
  }
}
```

## Validation

### Validation Process

1. **Syntax check**: Validate JSON-LD parses correctly (no trailing commas, proper quoting).
2. **Schema.org compliance**: Verify types and properties exist in Schema.org vocabulary.
3. **Google requirements**: Test with Google's Rich Results Test — it checks both validity and eligibility.
4. **Content match**: Ensure structured data matches visible page content (Google penalizes mismatched markup).

### Common Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Missing required field | `price` without `priceCurrency` | Add all required fields per schema type |
| Invalid enum value | `availability: "in stock"` | Use full URL: `https://schema.org/InStock` |
| Trailing comma | `"price": "29.99",}` | Remove trailing comma before closing brace |
| Mismatched content | Schema says $29.99 but page shows $39.99 | Sync structured data with displayed content |
| Self-serving reviews | Merchant-written reviews marked as customer reviews | Use only genuine customer reviews |

## Implementation Patterns

### Dynamic JSON-LD Generation

Generate structured data from application data rather than hardcoding:

```typescript
// Next.js App Router example
function ProductJsonLd({ product }: { product: Product }) {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "Product",
    name: product.name,
    image: product.images,
    description: product.description,
    offers: {
      "@type": "Offer",
      price: product.price.toString(),
      priceCurrency: "USD",
      availability: product.inStock
        ? "https://schema.org/InStock"
        : "https://schema.org/OutOfStock",
    },
  };

  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
    />
  );
}
```

### Edge Cases to Handle

- **Out-of-stock products**: Set `availability` to `https://schema.org/OutOfStock`, keep the product schema.
- **Price ranges**: Use `AggregateOffer` with `lowPrice` and `highPrice` instead of `Offer`.
- **Multiple variants**: Use `hasVariant` with individual `ProductModel` entries.
- **No reviews yet**: Omit `aggregateRating` entirely — do not set `ratingCount: 0`.

## References

- **[schema-type-reference.md](references/schema-type-reference.md)** — Complete reference for additional schema types including HowTo, Recipe, Event, VideoObject, SoftwareApplication, Course, and WebSite (Sitelinks Search Box). Includes required vs recommended fields, nested schema patterns, and Google-specific requirements for each type.
- **[rich-results-eligibility.md](references/rich-results-eligibility.md)** — Google Rich Results eligibility matrix showing which schema types produce which search features, content policies that can disqualify markup, testing and monitoring workflow with Search Console, and troubleshooting guide for structured data that validates but does not produce rich results.
