# UX and Accessibility Audit Tools

## Automated Accessibility Testing

### axe-core (Deque)

The industry standard for automated accessibility testing. Catches ~30-40% of WCAG issues.

```bash
# Install for CLI usage
npm install -g @axe-core/cli

# Run against a URL
axe https://example.com --tags wcag2a,wcag2aa

# Run against local dev server
axe http://localhost:3000 --save results.json
```

**Integration with testing frameworks:**

```javascript
// Playwright + axe
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('page has no accessibility violations', async ({ page }) => {
  await page.goto('/dashboard');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag22aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});

// Jest + Testing Library + axe
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

test('form is accessible', async () => {
  const { container } = render(<LoginForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### Lighthouse

Google's auditing tool for performance, accessibility, SEO, and best practices.

```bash
# CLI usage
npm install -g lighthouse
lighthouse https://example.com --output=json --output-path=./report.json

# Focus on accessibility only
lighthouse https://example.com --only-categories=accessibility

# CI integration with budget assertions
lighthouse https://example.com --budget-path=budget.json
```

**Budget file for CI:**

```json
[{
  "path": "/*",
  "timings": [
    { "metric": "interactive", "budget": 3000 },
    { "metric": "first-contentful-paint", "budget": 1500 }
  ],
  "resourceSizes": [
    { "resourceType": "script", "budget": 300 },
    { "resourceType": "total", "budget": 500 }
  ]
}]
```

### Pa11y

Lightweight accessibility testing with CI-friendly output.

```bash
# Install and run
npm install -g pa11y
pa11y https://example.com

# Run with specific standard
pa11y --standard WCAG2AA https://example.com

# CI-friendly JSON output
pa11y --reporter json https://example.com > results.json

# Test multiple pages
npm install -g pa11y-ci
pa11y-ci --config .pa11yci.json
```

**.pa11yci.json configuration:**

```json
{
  "defaults": {
    "standard": "WCAG2AA",
    "timeout": 10000,
    "wait": 1000
  },
  "urls": [
    "http://localhost:3000/",
    "http://localhost:3000/login",
    "http://localhost:3000/dashboard",
    {
      "url": "http://localhost:3000/form",
      "actions": ["set field #email to test@example.com", "click element #submit"]
    }
  ]
}
```

---

## Browser Extensions for UX Analysis

### Contrast and Color

| Tool | Purpose |
|------|---------|
| **WCAG Color Contrast Checker** | Real-time contrast ratio checking on any element |
| **Stark** | Contrast checking + color blindness simulation in browser |
| **Color Oracle** | System-wide color blindness simulator (desktop) |

### Layout and Spacing

| Tool | Purpose |
|------|---------|
| **VisBug** | Visual debugging: inspect spacing, alignment, typography, color |
| **CSS Grid Inspector** (DevTools) | Visualize grid lines, gap, areas |
| **Pesticide** | Outline every element to reveal layout issues |

### Performance

| Tool | Purpose |
|------|---------|
| **Web Vitals Extension** | Real-time CLS, LCP, FID/INP measurement |
| **Lighthouse DevTools** | Built-in performance profiling |
| **Performance Monitor** (DevTools) | CPU, memory, layout recalculations live |

---

## Screen Reader Testing

### VoiceOver (macOS/iOS) — Essential Commands

| Action | Shortcut |
|--------|----------|
| Turn on/off | `Cmd + F5` |
| Navigate next item | `VO + Right Arrow` (VO = Ctrl + Option) |
| Navigate previous item | `VO + Left Arrow` |
| Activate element | `VO + Space` |
| Read all from cursor | `VO + A` |
| Open rotor (navigate by type) | `VO + U` |
| Navigate headings | Use rotor → Headings |
| Navigate landmarks | Use rotor → Landmarks |
| Navigate form controls | Use rotor → Form Controls |
| Navigate links | Use rotor → Links |

### NVDA (Windows) — Essential Commands

| Action | Shortcut |
|--------|----------|
| Turn on | `Ctrl + Alt + N` |
| Stop speaking | `Ctrl` |
| Navigate next item | `Tab` (interactive) or `Down Arrow` (browse) |
| Navigate headings | `H` (next heading), `1-6` (heading level) |
| Navigate landmarks | `D` (next landmark) |
| Navigate form controls | `F` (next form field) |
| Navigate tables | `T` (next table), `Ctrl + Alt + Arrows` (cells) |
| List all headings | `NVDA + F7` |
| Read current line | `NVDA + L` |
| Elements list | `NVDA + F7` |

### Testing Script Template

```markdown
## Screen Reader Test: [Feature Name]

**Reader**: [NVDA / VoiceOver]
**Browser**: [Chrome / Safari / Firefox]

### Landmarks
- [ ] Page has main landmark
- [ ] Navigation landmarks labeled
- [ ] Complementary/aside landmarks present where expected

### Headings
- [ ] Heading hierarchy is logical (h1 → h2 → h3, no skips)
- [ ] Page title is h1
- [ ] Section headings accurately describe content

### Forms
- [ ] All inputs have accessible names
- [ ] Required fields announced
- [ ] Error messages associated with inputs (aria-describedby)
- [ ] Form validation errors announced via aria-live

### Dynamic Content
- [ ] Notifications announced (aria-live regions)
- [ ] Modal focus trapped correctly
- [ ] Focus returns to trigger after modal close
- [ ] Route changes announced
```

---

## Core Web Vitals and UX Impact

### Metrics and Their UX Meaning

| Metric | What It Measures | UX Impact | Good Threshold |
|--------|-----------------|-----------|----------------|
| **LCP** (Largest Contentful Paint) | When main content is visible | "Can I see the content?" | < 2.5s |
| **INP** (Interaction to Next Paint) | Input responsiveness | "Does it feel responsive?" | < 200ms |
| **CLS** (Cumulative Layout Shift) | Visual stability | "Did the page jump?" | < 0.1 |
| **FCP** (First Contentful Paint) | When first content appears | "Is anything happening?" | < 1.8s |
| **TTFB** (Time to First Byte) | Server response time | "Is the server alive?" | < 800ms |

### UX Decisions That Affect Metrics

| UX Decision | Metric Affected | Impact |
|------------|----------------|--------|
| Hero image without dimensions | CLS | Layout shift as image loads |
| Web font without fallback | LCP, CLS | Flash of invisible/unstyled text |
| Lazy-loaded above-fold content | LCP | Main content loads late |
| Unoptimized images | LCP | Large payload slows rendering |
| Heavy JS on interaction | INP | Clicks feel sluggish |
| Dynamic content injection | CLS | Content pushes elements down |
| Skeleton screens with correct dimensions | CLS | Prevents shift (good) |
| `loading="lazy"` on below-fold images | LCP | Reduces initial payload (good) |
| `fetchpriority="high"` on hero image | LCP | Prioritizes main content (good) |

### Measuring in DevTools

```javascript
// Performance Observer for real user monitoring
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.startTime.toFixed(0)}ms`);
  }
});

observer.observe({ type: 'largest-contentful-paint', buffered: true });
observer.observe({ type: 'layout-shift', buffered: true });
observer.observe({ type: 'first-input', buffered: true });
```

---

## Design Consistency Audit

### Token Usage Audit

To verify design tokens are applied consistently, audit the codebase for hard-coded values:

```bash
# Find hard-coded colors (hex, rgb, hsl not using variables)
grep -rn --include="*.css" --include="*.scss" '#[0-9a-fA-F]\{3,8\}' src/
grep -rn --include="*.css" 'rgb\|rgba\|hsl' src/ | grep -v 'var('

# Find hard-coded spacing (pixel values not using variables)
grep -rn --include="*.css" 'padding:\s*[0-9]' src/
grep -rn --include="*.css" 'margin:\s*[0-9]' src/
grep -rn --include="*.css" 'gap:\s*[0-9]' src/

# Find hard-coded font sizes
grep -rn --include="*.css" 'font-size:\s*[0-9]' src/
```

### Visual Regression Testing

```javascript
// Playwright visual comparison
test('button variants match design', async ({ page }) => {
  await page.goto('/storybook/button');
  await expect(page).toHaveScreenshot('button-variants.png', {
    maxDiffPixels: 50,
  });
});
```

---

## Data-Driven UX Analysis

### Key Interaction Metrics

| Metric | What It Reveals | Tool |
|--------|----------------|------|
| Task completion rate | Can users finish core tasks? | Analytics events |
| Time on task | How efficient is the flow? | Session recording |
| Error rate | Where do users make mistakes? | Form analytics |
| Rage clicks | Where are users frustrated? | Heatmap tools |
| Dead clicks | Where do users expect interactivity? | Heatmap tools |
| Scroll depth | Is important content being seen? | Scroll maps |
| Exit pages | Where do users abandon? | Analytics funnel |

### Implementing Key Events

```javascript
// Track task completion
analytics.track('task_completed', {
  task: 'checkout',
  steps: 3,
  duration_seconds: 45,
  errors_encountered: 0
});

// Track form errors
analytics.track('form_error', {
  form: 'signup',
  field: 'email',
  error: 'invalid_format',
  attempt: 2
});

// Track feature discovery
analytics.track('feature_discovered', {
  feature: 'keyboard_shortcuts',
  discovery_method: 'tooltip'  // tooltip, help_menu, accident
});
```
