# Schema Type Reference

## HowTo

Eligible for: step-by-step instructions with images in search results.

```json
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "How to Set Up Noise-Cancelling Headphones",
  "description": "A step-by-step guide to setting up your new noise-cancelling headphones.",
  "image": "https://example.com/images/howto-hero.jpg",
  "totalTime": "PT10M",
  "estimatedCost": {
    "@type": "MonetaryAmount",
    "currency": "USD",
    "value": "0"
  },
  "supply": [
    { "@type": "HowToSupply", "name": "Headphones" },
    { "@type": "HowToSupply", "name": "USB-C charging cable" }
  ],
  "tool": [
    { "@type": "HowToTool", "name": "Smartphone with Bluetooth" }
  ],
  "step": [
    {
      "@type": "HowToStep",
      "name": "Charge the headphones",
      "text": "Connect the USB-C cable and charge for at least 30 minutes before first use.",
      "image": "https://example.com/images/step1-charge.jpg",
      "url": "https://example.com/setup#step-charge"
    },
    {
      "@type": "HowToStep",
      "name": "Enable Bluetooth pairing mode",
      "text": "Press and hold the power button for 5 seconds until the LED flashes blue.",
      "image": "https://example.com/images/step2-pair.jpg",
      "url": "https://example.com/setup#step-pair"
    },
    {
      "@type": "HowToStep",
      "name": "Connect from your device",
      "text": "Open Bluetooth settings on your phone and select the headphones from the list.",
      "image": "https://example.com/images/step3-connect.jpg",
      "url": "https://example.com/setup#step-connect"
    }
  ]
}
```

**Required:** name, step (with text).
**Recommended:** image per step, totalTime, supply, tool.
**Rules:** Steps must match visible page content. Do not use for recipes (use Recipe type).

## Recipe

Eligible for: recipe card in search with image, rating, cook time, calories.

```json
{
  "@context": "https://schema.org",
  "@type": "Recipe",
  "name": "Classic Margherita Pizza",
  "image": [
    "https://example.com/photos/pizza-16x9.jpg",
    "https://example.com/photos/pizza-4x3.jpg",
    "https://example.com/photos/pizza-1x1.jpg"
  ],
  "author": {
    "@type": "Person",
    "name": "Chef Maria"
  },
  "datePublished": "2025-03-15",
  "description": "Authentic Neapolitan-style margherita pizza with San Marzano tomatoes and fresh mozzarella.",
  "prepTime": "PT30M",
  "cookTime": "PT15M",
  "totalTime": "PT45M",
  "recipeYield": "4 servings",
  "recipeCategory": "Main course",
  "recipeCuisine": "Italian",
  "nutrition": {
    "@type": "NutritionInformation",
    "calories": "285 calories",
    "fatContent": "12 g",
    "carbohydrateContent": "32 g",
    "proteinContent": "14 g"
  },
  "recipeIngredient": [
    "500g tipo 00 flour",
    "325ml warm water",
    "7g active dry yeast",
    "10g salt",
    "400g San Marzano tomatoes",
    "250g fresh mozzarella",
    "Fresh basil leaves",
    "Extra virgin olive oil"
  ],
  "recipeInstructions": [
    {
      "@type": "HowToStep",
      "text": "Combine flour, water, yeast, and salt. Knead for 10 minutes until smooth."
    },
    {
      "@type": "HowToStep",
      "text": "Let dough rise for 2 hours at room temperature."
    },
    {
      "@type": "HowToStep",
      "text": "Preheat oven to 500°F (260°C) with pizza stone."
    },
    {
      "@type": "HowToStep",
      "text": "Stretch dough, add crushed tomatoes, torn mozzarella, and basil."
    },
    {
      "@type": "HowToStep",
      "text": "Bake for 12-15 minutes until crust is golden and cheese is bubbly."
    }
  ],
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "ratingCount": "342"
  }
}
```

**Required:** name, image, recipeIngredient, recipeInstructions.
**Recommended:** author, datePublished, prepTime, cookTime, nutrition, aggregateRating.

## Event

Eligible for: event listing in search with date, location, ticket info.

```json
{
  "@context": "https://schema.org",
  "@type": "Event",
  "name": "Audio Technology Expo 2025",
  "description": "Annual exhibition showcasing the latest in audio technology.",
  "image": "https://example.com/images/expo-2025.jpg",
  "startDate": "2025-09-15T09:00:00-07:00",
  "endDate": "2025-09-17T18:00:00-07:00",
  "eventStatus": "https://schema.org/EventScheduled",
  "eventAttendanceMode": "https://schema.org/OfflineEventAttendanceMode",
  "location": {
    "@type": "Place",
    "name": "Convention Center",
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "123 Main St",
      "addressLocality": "San Francisco",
      "addressRegion": "CA",
      "postalCode": "94102",
      "addressCountry": "US"
    }
  },
  "organizer": {
    "@type": "Organization",
    "name": "AudioShop Events",
    "url": "https://example.com"
  },
  "offers": {
    "@type": "Offer",
    "url": "https://example.com/events/expo-2025/tickets",
    "price": "99.00",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "validFrom": "2025-06-01T00:00:00-07:00"
  },
  "performer": {
    "@type": "Person",
    "name": "Dr. Audio Expert"
  }
}
```

**Event status values:**
- `EventScheduled` — Event is happening as planned
- `EventPostponed` — Postponed (set new startDate when known)
- `EventCancelled` — Cancelled
- `EventMovedOnline` — Changed to virtual

**Attendance modes:**
- `OfflineEventAttendanceMode` — In-person
- `OnlineEventAttendanceMode` — Virtual
- `MixedEventAttendanceMode` — Hybrid

## VideoObject

Eligible for: video thumbnails and key moments in search results.

```json
{
  "@context": "https://schema.org",
  "@type": "VideoObject",
  "name": "Headphones Unboxing and Review",
  "description": "Detailed unboxing and 30-day review of the WH-1000XM5 headphones.",
  "thumbnailUrl": "https://example.com/thumbnails/review-video.jpg",
  "uploadDate": "2025-03-15T08:00:00+00:00",
  "duration": "PT12M30S",
  "contentUrl": "https://example.com/videos/headphones-review.mp4",
  "embedUrl": "https://example.com/embed/headphones-review",
  "hasPart": [
    {
      "@type": "Clip",
      "name": "Unboxing",
      "startOffset": 0,
      "endOffset": 120,
      "url": "https://example.com/videos/review#t=0"
    },
    {
      "@type": "Clip",
      "name": "Sound Quality Test",
      "startOffset": 120,
      "endOffset": 420,
      "url": "https://example.com/videos/review#t=120"
    },
    {
      "@type": "Clip",
      "name": "Battery Life Results",
      "startOffset": 420,
      "endOffset": 600,
      "url": "https://example.com/videos/review#t=420"
    }
  ]
}
```

**Required:** name, description, thumbnailUrl, uploadDate.
**Key moments (Clip):** Enable "key moments" in video search results showing chapter markers.

## SoftwareApplication

Eligible for: app information in search with rating, price, OS.

```json
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "AudioShop Mobile App",
  "operatingSystem": "Android, iOS",
  "applicationCategory": "ShoppingApplication",
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "USD"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.6",
    "ratingCount": "15200"
  }
}
```

## WebSite (Sitelinks Search Box)

Eligible for: search box within the sitelinks area of search results.

```json
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "AudioShop",
  "url": "https://example.com",
  "potentialAction": {
    "@type": "SearchAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "https://example.com/search?q={search_term_string}"
    },
    "query-input": "required name=search_term_string"
  }
}
```

**Placement:** Homepage only.
**Note:** The `urlTemplate` must match the actual search URL pattern.

## Course

Eligible for: course listing with provider, description, rating.

```json
{
  "@context": "https://schema.org",
  "@type": "Course",
  "name": "Audio Engineering Fundamentals",
  "description": "Learn the basics of audio engineering, mixing, and mastering.",
  "provider": {
    "@type": "Organization",
    "name": "AudioShop Academy",
    "sameAs": "https://example.com/academy"
  },
  "offers": {
    "@type": "Offer",
    "price": "49.99",
    "priceCurrency": "USD",
    "category": "Paid"
  },
  "hasCourseInstance": {
    "@type": "CourseInstance",
    "courseMode": "online",
    "courseSchedule": {
      "@type": "Schedule",
      "repeatCount": 12,
      "repeatFrequency": "Weekly"
    }
  }
}
```

## Nested Schema Patterns

### Product with Reviews (Deep Nesting)

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Product Name",
  "review": [
    {
      "@type": "Review",
      "author": { "@type": "Person", "name": "Reviewer" },
      "datePublished": "2025-03-01",
      "reviewRating": {
        "@type": "Rating",
        "ratingValue": "5",
        "bestRating": "5"
      },
      "reviewBody": "Review text here."
    }
  ],
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.7",
    "ratingCount": "1423",
    "bestRating": "5"
  }
}
```

### Using @id for Cross-References

When multiple schemas on a page reference the same entity:

```json
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Organization",
      "@id": "https://example.com/#organization",
      "name": "AudioShop",
      "url": "https://example.com"
    },
    {
      "@type": "Article",
      "publisher": { "@id": "https://example.com/#organization" },
      "headline": "Article Title"
    },
    {
      "@type": "BreadcrumbList",
      "itemListElement": [...]
    }
  ]
}
```

**Benefits of @id:** Avoids duplicating the same entity data across multiple schema blocks. Keeps JSON-LD clean and reduces size.

## Google-Specific Requirements

### Mandatory vs Recommended Properties

Google's requirements often exceed Schema.org's minimum specification. Always check:
- **Google Search Central documentation** for required fields per type.
- **Rich Results Test** to verify eligibility.
- **Content policies** — even valid schema is rejected if content violates policies.

### Common Google Content Policies

| Policy | Violation Example |
|--------|-------------------|
| Content mismatch | Schema shows $29.99 but page shows $39.99 |
| Self-serving reviews | Only positive, merchant-written reviews |
| Missing visible content | FAQ schema for questions not visible on page |
| Misleading markup | JobPosting for content that is not an actual job listing |
| Spammy practices | Auto-generated, irrelevant structured data |
