# XSS Prevention and Content Security Policy

Comprehensive reference covering React's escaping model, DOMPurify configuration, SSR-specific XSS risks, SVG injection, and Content Security Policy implementation for React and Next.js applications.

---

## React's Built-in Escaping

### How JSX Escaping Works

React escapes all values embedded in JSX before rendering to the DOM. Under the hood, `ReactDOM` converts special characters to HTML entities:

```
& → &amp;
< → &lt;
> → &gt;
" → &quot;
' → &#x27;
```

This means rendering `{userInput}` in JSX is inherently safe against XSS. The escaping is applied at the React element level, not at the string level.

### What JSX Escaping Does NOT Protect

1. **`dangerouslySetInnerHTML`** — bypasses all escaping
2. **`href` attributes** — `javascript:` and `data:` URIs execute code
3. **`src` attributes on `<iframe>`, `<script>`, `<object>`** — can load malicious content
4. **Style injection** — `style={{ background: userInput }}` can inject CSS expressions in old browsers
5. **Server-side rendering** — template injection before React hydration
6. **Spread props** — `<div {...userControlledObject} />` can inject any attribute

---

## DOMPurify Configuration

### Installation

```bash
npm install dompurify
npm install -D @types/dompurify
```

### Configuration Profiles

```typescript
import DOMPurify from 'dompurify';

// Strict: plain text with basic formatting
const STRICT_CONFIG: DOMPurify.Config = {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'br'],
  ALLOWED_ATTR: [],
};

// Moderate: rich content (blog posts, comments)
const MODERATE_CONFIG: DOMPurify.Config = {
  ALLOWED_TAGS: [
    'p', 'br', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li',
    'h2', 'h3', 'h4', 'blockquote', 'code', 'pre',
  ],
  ALLOWED_ATTR: ['href', 'title', 'target', 'rel'],
  ALLOW_DATA_ATTR: false,
  ADD_ATTR: ['target'],
};

// Custom hook for sanitized HTML
function useSanitizedHtml(html: string, config: DOMPurify.Config = MODERATE_CONFIG) {
  return { __html: DOMPurify.sanitize(html, config) };
}
```

### Preventing Attribute-Based Attacks

```typescript
// Hook into DOMPurify to force rel="noopener noreferrer" on all links
DOMPurify.addHook('afterSanitizeAttributes', (node) => {
  if (node.tagName === 'A') {
    node.setAttribute('target', '_blank');
    node.setAttribute('rel', 'noopener noreferrer');
  }
});
```

---

## URL Sanitization

### Protocol Allowlisting

```typescript
function sanitizeUrl(url: string): string | null {
  try {
    const parsed = new URL(url);
    if (!['http:', 'https:', 'mailto:'].includes(parsed.protocol)) {
      return null;
    }
    return parsed.href;
  } catch {
    return null;
  }
}

function SafeLink({ href, children }: { href: string; children: React.ReactNode }) {
  const safeHref = sanitizeUrl(href);
  if (!safeHref) {
    return <span>{children}</span>;
  }
  return (
    <a href={safeHref} rel="noopener noreferrer" target="_blank">
      {children}
    </a>
  );
}
```

### Image Source Validation

```typescript
function SafeImage({ src, alt }: { src: string; alt: string }) {
  const isSafe = /^https:\/\//i.test(src) || src.startsWith('/');
  if (!isSafe) return <div className="image-placeholder">{alt}</div>;
  return <img src={src} alt={alt} loading="lazy" />;
}
```

---

## SVG Injection Vectors

SVGs can contain embedded scripts and event handlers:

```html
<!-- Malicious SVG -->
<svg onload="alert('xss')">
  <text>Hello</text>
</svg>

<svg>
  <script>alert('xss')</script>
</svg>

<svg>
  <a xlink:href="javascript:alert('xss')">
    <text>Click me</text>
  </a>
</svg>
```

### Safe SVG Rendering

```typescript
import DOMPurify from 'dompurify';

const SVG_CONFIG: DOMPurify.Config = {
  USE_PROFILES: { svg: true },
  ALLOWED_TAGS: ['svg', 'path', 'circle', 'rect', 'line', 'polyline', 'polygon', 'g', 'text', 'defs', 'use'],
  ALLOWED_ATTR: [
    'd', 'fill', 'stroke', 'stroke-width', 'cx', 'cy', 'r', 'x', 'y',
    'width', 'height', 'viewBox', 'xmlns', 'transform', 'opacity',
  ],
};

function SafeSvg({ svgContent }: { svgContent: string }) {
  const sanitized = DOMPurify.sanitize(svgContent, SVG_CONFIG);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

---

## Server-Side Rendering XSS Risks

### Hydration Mismatch Injection

If server-rendered HTML contains unescaped user data, it executes before React hydrates:

```typescript
// DANGEROUS — raw string interpolation in SSR
function Page({ searchQuery }: { searchQuery: string }) {
  return (
    <html>
      <body>
        {/* This is safe — React escapes it */}
        <p>Results for: {searchQuery}</p>

        {/* DANGEROUS — if passed via raw HTML template */}
        <script
          dangerouslySetInnerHTML={{
            __html: `window.__INITIAL_STATE__ = ${JSON.stringify({ query: searchQuery })}`,
          }}
        />
      </body>
    </html>
  );
}
```

### Safe State Serialization

```typescript
import serialize from 'serialize-javascript';

// serialize-javascript escapes HTML-significant characters in strings
const safeState = serialize({ query: searchQuery }, { isJSON: true });

<script
  dangerouslySetInnerHTML={{
    __html: `window.__INITIAL_STATE__ = ${safeState}`,
  }}
/>
```

---

## Content Security Policy Implementation

### Nonce-Based CSP for Next.js

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');

  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`,
    `style-src 'self' 'nonce-${nonce}'`,
    `img-src 'self' data: https:`,
    `font-src 'self'`,
    `connect-src 'self' https://api.example.com wss://ws.example.com`,
    `media-src 'self'`,
    `object-src 'none'`,
    `frame-src 'none'`,
    `frame-ancestors 'none'`,
    `base-uri 'self'`,
    `form-action 'self'`,
    `upgrade-insecure-requests`,
  ].join('; ');

  const response = NextResponse.next();
  response.headers.set('Content-Security-Policy', csp);
  response.headers.set('x-nonce', nonce);
  return response;
}
```

### Passing Nonce to Components

```typescript
// app/layout.tsx
import { headers } from 'next/headers';
import Script from 'next/script';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const headerStore = await headers();
  const nonce = headerStore.get('x-nonce') ?? '';

  return (
    <html lang="en">
      <body>
        {children}
        <Script src="/analytics.js" nonce={nonce} strategy="afterInteractive" />
      </body>
    </html>
  );
}
```

### CSP Reporting

```typescript
// Add report-uri or report-to directive
const csp = [
  // ... other directives
  `report-uri /api/csp-report`,
  `report-to csp-endpoint`,
].join('; ');

// Report-To header (modern browsers)
response.headers.set(
  'Report-To',
  JSON.stringify({
    group: 'csp-endpoint',
    max_age: 86400,
    endpoints: [{ url: '/api/csp-report' }],
  })
);
```

```typescript
// app/api/csp-report/route.ts
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const report = await request.json();
  console.error('CSP Violation:', JSON.stringify(report, null, 2));
  // Forward to your logging/monitoring system
  return NextResponse.json({ received: true });
}
```

---

## Third-Party Component Risks

### Auditing Third-Party Libraries

Before using any component library that renders user content:

1. Check if it sanitizes HTML inputs internally
2. Look for known XSS CVEs in npm audit
3. Verify the library escapes attribute values
4. Test with common XSS payloads

### Common Vulnerable Patterns in Third-Party Components

```typescript
// Markdown renderers — verify they sanitize output
import ReactMarkdown from 'react-markdown';
import rehypeSanitize from 'rehype-sanitize';

// SAFE — with sanitization plugin
<ReactMarkdown rehypePlugins={[rehypeSanitize]}>
  {userMarkdown}
</ReactMarkdown>

// DANGEROUS — without sanitization (allows raw HTML in markdown)
<ReactMarkdown>
  {userMarkdown}
</ReactMarkdown>
```

### WYSIWYG Editors

Rich text editors like TipTap, Slate, or Quill produce HTML that must be sanitized before storage and before rendering:

```typescript
// Always sanitize on both input (before save) and output (before render)
function saveContent(html: string) {
  const sanitized = DOMPurify.sanitize(html, MODERATE_CONFIG);
  return api.savePost({ content: sanitized });
}

function renderContent(html: string) {
  const sanitized = DOMPurify.sanitize(html, MODERATE_CONFIG);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```
