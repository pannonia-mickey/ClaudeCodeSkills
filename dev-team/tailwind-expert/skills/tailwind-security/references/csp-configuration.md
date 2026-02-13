# CSP Configuration for Tailwind Projects

A comprehensive guide to configuring Content Security Policy headers for Tailwind CSS projects, covering CSP fundamentals, framework-specific integration, nonce propagation, third-party library handling, reporting, and testing strategies.

## CSP Fundamentals for CSS

Content Security Policy (CSP) is an HTTP response header that controls which resources a browser is allowed to load and execute. For CSS specifically, the `style-src` directive governs which stylesheets and inline styles the browser will apply.

### The `style-src` Directive

To control how styles are loaded, configure the `style-src` directive with one or more source expressions:

- **`'self'`**: Allow styles from the same origin as the document. This covers static CSS files served from the application's own domain, which is the default Tailwind output pattern.
- **`'unsafe-inline'`**: Allow all inline `<style>` tags and `style=""` attributes. Avoid this directive whenever possible, as it defeats the purpose of CSP for style injection attacks.
- **`'nonce-{value}'`**: Allow specific `<style>` tags that carry a matching `nonce` attribute. Generate a unique, cryptographically random nonce per request and include it in both the CSP header and the authorized `<style>` tags.
- **`'sha256-{hash}'`**: Allow inline style content that matches a specific SHA-256 hash. This is useful for static inline styles that never change, but impractical for dynamically generated style blocks.

To build a secure baseline for Tailwind projects, start with:

```
Content-Security-Policy: style-src 'self';
```

This permits the static Tailwind CSS file loaded via `<link>` and blocks all inline styles. To accommodate SSR-injected critical CSS, add a nonce:

```
Content-Security-Policy: style-src 'self' 'nonce-abc123def456';
```

### How Tailwind Fits the CSP Model

Tailwind CSS produces a static `.css` file at build time through the PostCSS pipeline. This file is served as a standard stylesheet via a `<link>` tag, which falls under `'self'` when served from the same origin. No runtime style injection occurs in a standard Tailwind build, making it inherently CSP-friendly.

To verify this in a project, inspect the built HTML output and confirm that no `<style>` tags are injected by Tailwind itself. Any inline styles present will originate from the framework, UI libraries, or application code rather than from Tailwind's build output.

## Next.js + Tailwind CSP Setup

Next.js applications using Tailwind CSS require nonce-based CSP to accommodate Server-Side Rendering, which may inject `<style>` tags for critical CSS. To implement this, create a middleware that generates a per-request nonce and attaches it to the CSP header.

### Complete Middleware Example

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
    `connect-src 'self'`,
    `frame-ancestors 'none'`,
    `base-uri 'self'`,
    `form-action 'self'`,
  ].join('; ');

  const response = NextResponse.next();
  response.headers.set('Content-Security-Policy', csp);
  response.headers.set('x-nonce', nonce);
  return response;
}
```

To make the nonce available to components that need to inject authorized `<style>` tags, access it via the request headers in Server Components:

```typescript
// app/layout.tsx
import { headers } from 'next/headers';

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const headersList = await headers();
  const nonce = headersList.get('x-nonce') ?? '';

  return (
    <html lang="en">
      <head>
        <meta property="csp-nonce" content={nonce} />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

To ensure all framework-injected style tags carry the nonce, pass it through the rendering context. In Next.js App Router, the framework automatically applies the nonce to its own `<style>` tags when the `x-nonce` header is present.

## Nonce Propagation in SSR

Server-Side Rendering frameworks inject `<style>` tags into the HTML response for critical CSS, requiring nonce propagation to satisfy CSP policies.

### Generating Per-Request Nonces

To generate a secure nonce, use a cryptographically random value for each request. Never reuse nonces across requests, as this defeats their purpose:

```typescript
// Node.js / Edge Runtime
const nonce = Buffer.from(crypto.randomUUID()).toString('base64');

// Alternative: crypto.getRandomValues
const array = new Uint8Array(16);
crypto.getRandomValues(array);
const nonce = Buffer.from(array).toString('base64');
```

To ensure uniqueness, generate the nonce as early as possible in the request lifecycle (middleware or server handler) and propagate it through the rendering pipeline.

### Passing Nonce to Style Tags in SSR Output

To authorize SSR-injected style tags, attach the nonce attribute to every `<style>` element in the server-rendered HTML:

```html
<style nonce="abc123def456">
  /* Critical CSS injected by framework */
  .container { max-width: 1200px; }
</style>
```

### Framework-Specific Nonce Injection

**Next.js (App Router)**: Access the nonce via `headers()` in Server Components and pass it to any component that renders `<style>` tags. The framework handles nonce injection for its own critical CSS when the `x-nonce` header is set in middleware.

**Vite / Nuxt**: To inject nonces in Vite-based projects, use server middleware to intercept the HTML response and modify `<style>` tags before sending them to the client:

```typescript
// Vite plugin for nonce injection
function noncePlugin(): Plugin {
  return {
    name: 'nonce-inject',
    transformIndexHtml(html, ctx) {
      const nonce = ctx.server?.config?.nonce ?? generateNonce();
      return html.replace(
        /<style([\s>])/g,
        `<style nonce="${nonce}"$1`
      );
    },
  };
}
```

**Nuxt**: To configure CSP with nonces in Nuxt, use the `useHead` composable along with server middleware that generates and propagates the nonce through the Nuxt rendering context:

```typescript
// server/middleware/csp.ts
export default defineEventHandler((event) => {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  event.context.nonce = nonce;
  setHeader(event, 'Content-Security-Policy',
    `style-src 'self' 'nonce-${nonce}'; script-src 'self' 'nonce-${nonce}'`
  );
});
```

## Handling Third-Party Style Injection

Third-party libraries often inject inline styles at runtime, which conflicts with strict CSP policies. To address this without weakening the entire CSP configuration, identify the source of each violation and apply targeted solutions.

### UI Libraries That Inject Inline Styles

Libraries such as Radix UI and Headless UI may apply inline `style` attributes for positioning (e.g., popover placement, scroll locking). To handle these:

- Audit the library's CSP documentation for recommended configuration
- Check if the library supports a `nonce` prop or CSP-compatible mode
- For positioning styles, some libraries offer CSS custom property alternatives that work without inline styles

To determine whether a library injects inline styles, inspect the rendered DOM in browser DevTools and look for `style=""` attributes on library-generated elements.

### CSS-in-JS Libraries Used Alongside Tailwind

When a project uses styled-components, Emotion, or other CSS-in-JS libraries alongside Tailwind, those libraries inject `<style>` tags at runtime:

- To authorize these style tags, pass the CSP nonce to the CSS-in-JS provider:

```typescript
// styled-components with nonce
import { StyleSheetManager } from 'styled-components';

function App({ nonce }: { nonce: string }) {
  return (
    <StyleSheetManager nonce={nonce}>
      <AppContent />
    </StyleSheetManager>
  );
}

// Emotion with nonce
import { CacheProvider } from '@emotion/react';
import createCache from '@emotion/cache';

const cache = createCache({ key: 'css', nonce });
function App() {
  return (
    <CacheProvider value={cache}>
      <AppContent />
    </CacheProvider>
  );
}
```

- To minimize the CSP surface area, contain CSS-in-JS usage to the smallest possible scope and prefer Tailwind utilities for the majority of styling
- As a last resort, add the library's domain to `style-src` if it loads external stylesheets, but avoid `'unsafe-inline'` whenever a nonce-based solution exists

## CSP Reporting

To monitor CSP violations before and after enforcement, configure reporting directives that send violation data to a collection endpoint.

### Report-Only Mode

To test a CSP policy without blocking any resources, deploy it in report-only mode first:

```
Content-Security-Policy-Report-Only: style-src 'self' 'nonce-{value}'; report-uri /csp-report;
```

This header logs violations without enforcing the policy, allowing identification of all violating resources before switching to enforcement mode. To transition to enforcement, replace `Content-Security-Policy-Report-Only` with `Content-Security-Policy` once all violations are resolved.

### Configuring Report Endpoints

To collect violation reports, use the `report-uri` directive (widely supported) or the newer `report-to` directive:

```
Content-Security-Policy: style-src 'self'; report-uri /api/csp-report;
```

For the `report-to` directive, define a reporting endpoint in the `Reporting-Endpoints` header:

```
Reporting-Endpoints: csp-endpoint="/api/csp-report"
Content-Security-Policy: style-src 'self'; report-to csp-endpoint;
```

To handle incoming reports on the server:

```typescript
// /api/csp-report route handler
export async function POST(request: Request) {
  const report = await request.json();
  console.log('CSP Violation:', {
    blockedUri: report['csp-report']?.['blocked-uri'],
    violatedDirective: report['csp-report']?.['violated-directive'],
    documentUri: report['csp-report']?.['document-uri'],
  });
  // Forward to logging service (Sentry, Datadog, etc.)
  return new Response(null, { status: 204 });
}
```

### Monitoring Services

To aggregate and analyze CSP reports at scale, integrate with dedicated monitoring services:

- **report-uri.com**: Dedicated CSP report collection and analysis dashboard
- **Sentry**: CSP monitoring via the security reporting feature; configure the report URI to point to the Sentry CSP endpoint
- **Datadog / New Relic**: Forward CSP reports to application monitoring platforms for correlation with other telemetry

To set up Sentry CSP monitoring, use the project's CSP report endpoint provided in the Sentry dashboard:

```
Content-Security-Policy: style-src 'self'; report-uri https://sentry.io/api/{project}/security/?sentry_key={key};
```

## Tailwind Play CDN Security Warning

The `@tailwindcss/browser` package (formerly Tailwind Play CDN) generates and injects styles at runtime in the browser. This fundamentally conflicts with CSP because it requires `style-src 'unsafe-inline'`, which weakens the entire CSP policy.

To understand the risk:

- The Play CDN evaluates class names in the browser and creates corresponding `<style>` tags dynamically
- Requiring `'unsafe-inline'` opens the door to any inline style injection, not just Tailwind's
- There is no nonce-based workaround because the styles are generated client-side at unpredictable times

To handle this appropriately:

- Use `@tailwindcss/browser` only for prototyping, demos, and documentation examples
- Never deploy it to production environments that serve real users
- To migrate from Play CDN to a production build, install Tailwind CSS as a PostCSS plugin or use the Tailwind CLI to generate a static CSS file
- Add a CI check that flags imports of `@tailwindcss/browser` in production code paths

## Full Security Headers

To provide defense in depth alongside CSP, configure the complete set of recommended security headers:

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{value}' 'strict-dynamic'; style-src 'self' 'nonce-{value}'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; report-uri /api/csp-report;
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

To understand each header's role:

- **`X-Content-Type-Options: nosniff`**: Prevent browsers from MIME-sniffing responses, ensuring CSS files are treated as CSS
- **`X-Frame-Options: DENY`**: Prevent the page from being embedded in iframes, blocking clickjacking attacks
- **`X-XSS-Protection: 0`**: Disable the legacy XSS auditor, which can introduce vulnerabilities in modern browsers; rely on CSP instead
- **`Referrer-Policy: strict-origin-when-cross-origin`**: Limit referrer information sent to external origins, reducing data leakage
- **`Permissions-Policy`**: Restrict access to sensitive browser APIs that the application does not use
- **`Strict-Transport-Security`**: Enforce HTTPS connections for the domain and all subdomains

To apply these headers in a Next.js project, add them to `next.config.js`:

```typescript
// next.config.js
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-XSS-Protection', value: '0' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains' },
];

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```

To handle CSP separately (since it requires per-request nonce generation), keep the CSP header in middleware and the static headers in `next.config.js`.

## Testing CSP

To validate that the CSP configuration works correctly and does not break application functionality, test at multiple levels.

### Browser DevTools Console

To identify CSP violations during development, open the browser DevTools Console and look for messages prefixed with `[Report Only]` or `Refused to apply`:

```
Refused to apply inline style because it violates the following
Content Security Policy directive: "style-src 'self' 'nonce-abc123'".
```

To test efficiently, navigate through all application routes and interact with dynamic components (modals, dropdowns, tooltips) to trigger any lazy-loaded style injections.

### Google CSP Evaluator

To analyze a CSP policy for common weaknesses, submit it to `csp-evaluator.withgoogle.com`. The tool checks for:

- Use of `'unsafe-inline'` or `'unsafe-eval'`
- Overly broad source expressions (e.g., `https:`)
- Missing directives that fall back to `default-src`
- Known bypasses for the configured policy

To use it, paste the full CSP header value into the evaluator and review each directive's rating.

### Automated Testing with Playwright

To verify CSP compliance in CI, write Playwright tests that check for the absence of console errors related to CSP violations:

```typescript
import { test, expect } from '@playwright/test';

test('page renders without CSP violations', async ({ page }) => {
  const cspViolations: string[] = [];

  page.on('console', (msg) => {
    if (msg.text().includes('Content Security Policy')) {
      cspViolations.push(msg.text());
    }
  });

  await page.goto('/');
  await page.waitForLoadState('networkidle');

  // Interact with dynamic elements to trigger potential violations
  await page.click('[data-testid="open-modal"]');
  await page.waitForSelector('[data-testid="modal-content"]');

  expect(cspViolations).toEqual([]);
});

test('CSP header is present and correct', async ({ request }) => {
  const response = await request.get('/');
  const csp = response.headers()['content-security-policy'];

  expect(csp).toBeDefined();
  expect(csp).toContain("style-src 'self'");
  expect(csp).not.toContain("'unsafe-inline'");
  expect(csp).toContain('nonce-');
});
```

### CI Integration

To integrate CSP testing into the CI pipeline:

1. Run the application server with production CSP headers enabled
2. Execute Playwright tests against the running server
3. Fail the build if any CSP violations are detected
4. Parse CSP report-only logs for new violations introduced by the current changeset

To set up a GitHub Actions workflow for CSP testing:

```yaml
# .github/workflows/csp-test.yml
name: CSP Testing
on: [pull_request]
jobs:
  csp-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build
      - run: npm start &
      - run: npx wait-on http://localhost:3000
      - run: npx playwright test tests/csp/
```

To maintain CSP compliance over time, run these tests on every pull request and block merges that introduce new violations. Combine automated testing with periodic manual review of CSP reports to catch edge cases that automated tests may miss.
