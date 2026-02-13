# Meta Tag Patterns Reference

## Complete Meta Tag Inventory

### Essential Meta Tags (Every Page)

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Primary Keyword — Secondary Keyword | Brand</title>
  <meta name="description" content="150-160 character description with primary keyword and value proposition." />
  <link rel="canonical" href="https://example.com/current-page" />
</head>
```

### Robot Control Tags

```html
<!-- Standard indexing control -->
<meta name="robots" content="index, follow" />

<!-- Per-engine targeting -->
<meta name="googlebot" content="index, follow, max-snippet:-1, max-image-preview:large" />
<meta name="bingbot" content="index, follow" />

<!-- Content freshness for news/time-sensitive -->
<meta name="robots" content="unavailable_after: 2025-12-31T23:59:59+00:00" />
```

### Social and Sharing Tags

```html
<!-- Open Graph (Facebook, LinkedIn, Discord, Slack) -->
<meta property="og:type" content="website" />
<meta property="og:title" content="Page Title" />
<meta property="og:description" content="Page description for social sharing." />
<meta property="og:image" content="https://example.com/og-image.jpg" />
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta property="og:image:alt" content="Description of the image" />
<meta property="og:url" content="https://example.com/page" />
<meta property="og:site_name" content="Brand Name" />
<meta property="og:locale" content="en_US" />

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:site" content="@brandhandle" />
<meta name="twitter:creator" content="@authorhandle" />
<meta name="twitter:title" content="Page Title" />
<meta name="twitter:description" content="Page description for Twitter." />
<meta name="twitter:image" content="https://example.com/twitter-image.jpg" />
<meta name="twitter:image:alt" content="Description of the image" />
```

### Additional Useful Tags

```html
<!-- Prevent Google from showing a different title -->
<meta name="google" content="notranslate" />

<!-- Theme color for mobile browsers -->
<meta name="theme-color" content="#1a73e8" />

<!-- Referrer policy -->
<meta name="referrer" content="strict-origin-when-cross-origin" />

<!-- Verification tags -->
<meta name="google-site-verification" content="verification_token" />
<meta name="msvalidate.01" content="bing_verification_token" />
```

## Page Type Templates

### Homepage

```html
<title>Brand Name — Tagline or Primary Value Proposition</title>
<meta name="description" content="Brand description with primary service/product keywords. Include location if local. 150-160 chars." />
<meta property="og:type" content="website" />
```

- Title: Brand-focused, 50-60 characters.
- Description: Broad value proposition, primary keywords.
- Schema: Organization + WebSite (for Sitelinks Search Box).

### Product Page

```html
<title>Product Name — Key Feature | Brand</title>
<meta name="description" content="Buy Product Name with Key Feature. Price from $X. Free shipping. Read N reviews. Brand." />
<meta property="og:type" content="product" />
<meta property="product:price:amount" content="29.99" />
<meta property="product:price:currency" content="USD" />
```

- Title: Product name with differentiating feature, brand last.
- Description: Price, shipping, reviews — CTR-driving information.
- Schema: Product with Offer, AggregateRating, BreadcrumbList.

### Blog Article

```html
<title>Article Title — Specific and Compelling | Brand Blog</title>
<meta name="description" content="Summary of article value. What the reader will learn. Include primary keyword naturally." />
<meta property="og:type" content="article" />
<meta property="article:published_time" content="2025-03-15T08:00:00Z" />
<meta property="article:modified_time" content="2025-04-01T10:30:00Z" />
<meta property="article:author" content="https://example.com/authors/name" />
<meta property="article:section" content="Technology" />
<meta property="article:tag" content="headphones" />
```

- Title: Specific, compelling headline (not clickbait).
- Description: What the reader will learn or gain.
- Schema: Article with author, publisher, datePublished, dateModified.

### Category / Collection Page

```html
<title>Category Name — Browse N Products | Brand</title>
<meta name="description" content="Shop Category Name at Brand. Browse N products with free shipping. Filter by feature, price, rating." />
<meta property="og:type" content="website" />
```

- Title: Category name with product count or differentiator.
- Description: Selection size, filters, shipping — shopping intent.
- Schema: BreadcrumbList, optional CollectionPage.

### Local Business Page

```html
<title>Brand Name City — Service | Address</title>
<meta name="description" content="Visit Brand Name in City. Services include X, Y, Z. Open Mon-Sat 9am-6pm. Call (555) 123-4567." />
<meta property="og:type" content="business.business" />
```

- Title: Brand + City + primary service.
- Description: Address, hours, phone — local search intent.
- Schema: LocalBusiness with address, hours, geo coordinates.

## Framework-Specific Implementations

### Next.js App Router (generateMetadata)

```typescript
import type { Metadata, ResolvingMetadata } from 'next';

interface Props {
  params: { slug: string };
}

export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const product = await getProduct(params.slug);

  return {
    title: `${product.name} — ${product.tagline} | Brand`,
    description: product.metaDescription,
    alternates: {
      canonical: `https://example.com/products/${params.slug}`,
    },
    openGraph: {
      title: product.name,
      description: product.description,
      images: [
        {
          url: product.ogImage,
          width: 1200,
          height: 630,
          alt: product.name,
        },
      ],
      type: 'website',
    },
    twitter: {
      card: 'summary_large_image',
      title: product.name,
      description: product.description,
      images: [product.twitterImage],
    },
    robots: {
      index: product.isPublished,
      follow: true,
    },
  };
}
```

### Nuxt 3 (useSeoMeta)

```typescript
<script setup lang="ts">
const product = await useAsyncData('product', () =>
  $fetch(`/api/products/${route.params.slug}`)
);

useSeoMeta({
  title: () => `${product.value.name} — ${product.value.tagline} | Brand`,
  description: () => product.value.metaDescription,
  ogTitle: () => product.value.name,
  ogDescription: () => product.value.description,
  ogImage: () => product.value.ogImage,
  ogType: 'website',
  twitterCard: 'summary_large_image',
  twitterTitle: () => product.value.name,
  twitterDescription: () => product.value.description,
  twitterImage: () => product.value.twitterImage,
});

useHead({
  link: [
    {
      rel: 'canonical',
      href: `https://example.com/products/${route.params.slug}`,
    },
  ],
});
</script>
```

### Remix (meta function)

```typescript
import type { MetaFunction, LoaderFunctionArgs } from '@remix-run/node';

export const meta: MetaFunction<typeof loader> = ({ data }) => {
  if (!data?.product) {
    return [{ title: 'Product Not Found' }];
  }

  return [
    { title: `${data.product.name} | Brand` },
    { name: 'description', content: data.product.metaDescription },
    { property: 'og:title', content: data.product.name },
    { property: 'og:description', content: data.product.description },
    { property: 'og:image', content: data.product.ogImage },
    { property: 'og:type', content: 'website' },
    { name: 'twitter:card', content: 'summary_large_image' },
    {
      tagName: 'link',
      rel: 'canonical',
      href: `https://example.com/products/${data.product.slug}`,
    },
  ];
};
```

### Astro (Frontmatter / Layout)

```astro
---
// src/pages/products/[slug].astro
import BaseLayout from '../../layouts/BaseLayout.astro';

const { slug } = Astro.params;
const product = await getProduct(slug);

const seo = {
  title: `${product.name} — ${product.tagline} | Brand`,
  description: product.metaDescription,
  canonical: `https://example.com/products/${slug}`,
  ogImage: product.ogImage,
};
---

<BaseLayout {seo}>
  <h1>{product.name}</h1>
  <!-- content -->
</BaseLayout>
```

```astro
---
// src/layouts/BaseLayout.astro
const { title, description, canonical, ogImage } = Astro.props.seo;
---

<html>
  <head>
    <title>{title}</title>
    <meta name="description" content={description} />
    <link rel="canonical" href={canonical} />
    <meta property="og:title" content={title} />
    <meta property="og:description" content={description} />
    <meta property="og:image" content={ogImage} />
    <meta property="og:type" content="website" />
  </head>
  <body>
    <slot />
  </body>
</html>
```

## Title Tag Optimization Checklist

- [ ] 50-60 characters (check pixel width: ~580px max)
- [ ] Primary keyword within first 3-4 words
- [ ] Unique across all pages on the site
- [ ] Brand name at end (unless brand is the keyword)
- [ ] No keyword stuffing or repetition
- [ ] Compelling for CTR — reads naturally to humans
- [ ] Matches search intent (informational, commercial, transactional)
- [ ] Does not start with brand name (except homepage)
- [ ] Uses separator: pipe `|`, dash `—`, or colon `:`
