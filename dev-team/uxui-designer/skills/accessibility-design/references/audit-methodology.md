# Accessibility Audit Methodology Reference

## Automated Testing Setup

### axe-core CI Integration

```javascript
// playwright.config.ts — accessibility test suite
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests/a11y',
  use: {
    baseURL: 'http://localhost:3000',
  },
});

// tests/a11y/pages.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

const pages = [
  { name: 'Home', path: '/' },
  { name: 'Login', path: '/login' },
  { name: 'Dashboard', path: '/dashboard' },
  { name: 'Settings', path: '/settings' },
];

for (const page of pages) {
  test(`${page.name} has no accessibility violations`, async ({ page: pw }) => {
    await pw.goto(page.path);
    const results = await new AxeBuilder({ page: pw })
      .withTags(['wcag2a', 'wcag2aa', 'wcag22aa'])
      .exclude('.third-party-widget') // Exclude elements you can't control
      .analyze();

    expect(results.violations).toEqual([]);
  });
}
```

### GitHub Actions Workflow

```yaml
name: Accessibility Tests
on: [push, pull_request]

jobs:
  a11y:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci
      - run: npm run build
      - run: npm start &
      - run: npx wait-on http://localhost:3000

      - name: Run axe accessibility tests
        run: npx playwright test tests/a11y/

      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v11
        with:
          urls: |
            http://localhost:3000/
            http://localhost:3000/login
          budgetPath: ./lighthouse-budget.json
```

### Component-Level Testing

```javascript
// In unit tests — test each component
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

describe('Button accessibility', () => {
  test('default button is accessible', async () => {
    const { container } = render(<Button>Click me</Button>);
    expect(await axe(container)).toHaveNoViolations();
  });

  test('disabled button is accessible', async () => {
    const { container } = render(<Button disabled>Click me</Button>);
    expect(await axe(container)).toHaveNoViolations();
  });

  test('icon-only button requires aria-label', async () => {
    const { container } = render(
      <Button aria-label="Close"><CloseIcon /></Button>
    );
    expect(await axe(container)).toHaveNoViolations();
  });
});
```

---

## Manual Testing Procedures

### Keyboard-Only Walkthrough

Disconnect or cover your mouse. Navigate the entire application using only keyboard.

**Checklist for each page:**

1. **Tab order**
   - [ ] Tab moves through all interactive elements in logical order
   - [ ] No elements are skipped or reached out of visual order
   - [ ] Focus never gets trapped (except in modals)
   - [ ] Can reach all functionality without a mouse

2. **Focus visibility**
   - [ ] Focus indicator is always visible on the focused element
   - [ ] Focus indicator has sufficient contrast (3:1 minimum)
   - [ ] Focus is never hidden behind sticky headers or footers

3. **Interaction**
   - [ ] Buttons activate with Enter and Space
   - [ ] Links activate with Enter
   - [ ] Dropdowns/selects respond to Arrow keys
   - [ ] Escape closes modals, menus, and popups
   - [ ] Custom components have documented keyboard patterns

4. **Focus management**
   - [ ] Opening a modal moves focus into the modal
   - [ ] Closing a modal returns focus to the trigger
   - [ ] Route changes move focus to the page heading or main content
   - [ ] Deleted items move focus to the next item or container

### Screen Reader Testing Protocol

Test with at least two combinations from the primary testing matrix:

| Screen Reader | Browser | Priority |
|--------------|---------|----------|
| **NVDA** | Chrome | High (Windows primary) |
| **VoiceOver** | Safari | High (macOS/iOS primary) |
| **JAWS** | Chrome | Medium (enterprise) |
| **TalkBack** | Chrome | Medium (Android) |
| **VoiceOver** | Mobile Safari | Medium (iOS) |

### Screen Reader Testing Script

For each page, verify:

```markdown
## Screen Reader Test: [Page Name]

### Landmarks
- [ ] `main` landmark present and labeled
- [ ] `navigation` landmarks labeled distinctly
- [ ] `banner` (header) present
- [ ] `contentinfo` (footer) present
- [ ] Can navigate between landmarks (NVDA: D key, VO: rotor)

### Headings
- [ ] h1 present and describes the page
- [ ] Heading hierarchy logical (no skipped levels)
- [ ] Can navigate by headings (NVDA: H key, VO: rotor)
- [ ] Headings accurately describe their sections

### Forms
- [ ] Each input announced with its label
- [ ] Required fields announced as "required"
- [ ] Error messages read when navigating to the field
- [ ] Form instructions read before the first field
- [ ] Submit result announced (success/error)

### Images
- [ ] Informative images have descriptive alt text
- [ ] Decorative images have empty alt (alt="") or are hidden
- [ ] Complex images have extended descriptions

### Dynamic Content
- [ ] Toast notifications announced
- [ ] Loading states announced ("Loading..." / "Content loaded")
- [ ] Error alerts announced immediately
- [ ] List count changes announced ("5 results found")

### Tables
- [ ] Column headers associated with cells
- [ ] Can navigate cells with Ctrl+Alt+Arrows
- [ ] Sort state announced on column headers
```

---

## Common Screen Reader Commands for Testing

### NVDA Quick Reference

| Action | Command |
|--------|---------|
| Start reading | `NVDA + Down Arrow` |
| Stop reading | `Ctrl` |
| Next heading | `H` |
| Heading level 1-6 | `1` through `6` |
| Next landmark | `D` |
| Next form field | `F` |
| Next button | `B` |
| Next link | `K` (unvisited) / `V` (visited) |
| Next table | `T` |
| Table cell navigation | `Ctrl + Alt + Arrow Keys` |
| Next list | `L` |
| Elements list | `NVDA + F7` |
| Toggle forms mode | `NVDA + Space` |

### VoiceOver Quick Reference (macOS)

| Action | Command |
|--------|---------|
| Start VoiceOver | `Cmd + F5` |
| Next item | `VO + Right Arrow` |
| Previous item | `VO + Left Arrow` |
| Activate | `VO + Space` |
| Open rotor | `VO + U` |
| Navigate in rotor | `Arrow Keys` |
| Read all | `VO + A` |
| Stop reading | `Ctrl` |
| Navigate headings | Rotor → Headings → Arrows |
| Navigate landmarks | Rotor → Landmarks → Arrows |
| Web spots | `VO + Cmd + N` (next) / `VO + Cmd + P` (previous) |

`VO` = `Ctrl + Option`

---

## Remediation Prioritization Framework

### Impact-Frequency-Severity Matrix

Score each issue on three dimensions:

| Dimension | 1 (Low) | 2 (Medium) | 3 (High) |
|-----------|---------|------------|----------|
| **Impact** | Cosmetic, mild inconvenience | Significant barrier, workaround exists | Complete blocker, no workaround |
| **Frequency** | Edge case, rare user path | Common feature, some users affected | Core flow, all users affected |
| **User base** | Affects narrow disability category | Affects multiple disability categories | Affects broad population (including situational) |

**Priority Score** = Impact x Frequency x User Base

| Score | Priority | Action |
|-------|----------|--------|
| 18-27 | P0 — Critical | Fix immediately, blocks release |
| 8-17 | P1 — High | Fix in current sprint |
| 4-7 | P2 — Medium | Fix in next sprint |
| 1-3 | P3 — Low | Backlog |

### Common P0 Issues (Always Fix Immediately)

- No keyboard access to core functionality
- Missing form labels (screen readers cannot identify inputs)
- Focus trap with no escape
- Auto-playing media with no pause control
- Color as the only means of conveying information
- Contrast ratio below 3:1 on interactive elements
- Missing alt text on informative images in core flow

---

## Accessibility Statement Template

```markdown
# Accessibility Statement

## Commitment
[Company name] is committed to ensuring digital accessibility for people
with disabilities. We continually improve the user experience for everyone
and apply the relevant accessibility standards.

## Standards
We aim to conform to WCAG 2.2 Level AA.

## Current Status
[Date of last audit]: [Summary of compliance status]

### Known Issues
| Issue | Impact | Status | Expected Fix |
|-------|--------|--------|-------------|
| [Description] | [Who is affected] | [In progress / Planned] | [Date] |

## Testing
We test with the following assistive technologies:
- NVDA with Chrome on Windows
- VoiceOver with Safari on macOS
- TalkBack with Chrome on Android
- VoiceOver with Safari on iOS

## Feedback
If you encounter accessibility barriers, please contact us:
- Email: accessibility@example.com
- Phone: [number]
- Response time: [X business days]
```

---

## Color Contrast Testing

### Systematic Process

1. **Automated scan**: Run axe-core or Lighthouse for bulk contrast checking
2. **Manual review of dynamic states**: Check hover, focus, active, disabled states (tools often miss these)
3. **Check overlays**: Text on images, gradients, semi-transparent backgrounds
4. **Check placeholder text**: Often fails contrast (use visible labels instead)
5. **Check disabled states**: While not required by WCAG, visible disabled states improve usability

### Tools

| Tool | Use Case |
|------|----------|
| **axe DevTools** | Automated scanning in browser |
| **Colour Contrast Analyser (CCA)** | Desktop app for eyedropper-based checking |
| **WebAIM Contrast Checker** | Quick web-based ratio calculation |
| **Stark (Figma/browser)** | Design tool integration |
| **Polypane** | Browser with built-in contrast overlay |

---

## Mobile Accessibility Testing

### TalkBack (Android) Key Commands

| Action | Gesture |
|--------|---------|
| Next item | Swipe right |
| Previous item | Swipe left |
| Activate | Double tap |
| Scroll | Two-finger swipe |
| Open context menu | Swipe up then right |
| Navigate by heading | Swipe up/down to change navigation mode, then swipe right/left |

### iOS VoiceOver Key Gestures

| Action | Gesture |
|--------|---------|
| Next item | Swipe right |
| Previous item | Swipe left |
| Activate | Double tap |
| Scroll | Three-finger swipe |
| Open rotor | Two-finger rotation |
| Navigate by rotor category | Swipe up/down |
| Dismiss alert | Two-finger scrub (Z shape) |

### Mobile-Specific Checks

- [ ] Touch targets at least 44x44px
- [ ] No functionality dependent on hover
- [ ] Pinch-to-zoom not disabled (`user-scalable=no` must not be set)
- [ ] Landscape and portrait both functional
- [ ] Content accessible at 200% text zoom
- [ ] No horizontal scroll at 320px viewport width

---

## Regression Testing Strategy

### Preventing Accessibility Regressions

1. **Component-level axe tests**: Every component has a `toHaveNoViolations` test
2. **Page-level tests in CI**: Critical pages tested on every PR
3. **Keyboard navigation tests**: Playwright tests that verify tab order and key interactions
4. **Visual regression for focus indicators**: Screenshot tests that capture focus states
5. **Storybook a11y addon**: Real-time accessibility checking during development

```javascript
// Playwright keyboard regression test
test('modal focus trap works correctly', async ({ page }) => {
  await page.goto('/page-with-modal');
  await page.click('[data-testid="open-modal"]');

  // Focus should be inside modal
  const focusedElement = await page.evaluate(() => document.activeElement?.closest('[role="dialog"]'));
  expect(focusedElement).toBeTruthy();

  // Tab should cycle within modal
  await page.keyboard.press('Tab');
  await page.keyboard.press('Tab');
  await page.keyboard.press('Tab');
  const stillInModal = await page.evaluate(() => document.activeElement?.closest('[role="dialog"]'));
  expect(stillInModal).toBeTruthy();

  // Escape should close and restore focus
  await page.keyboard.press('Escape');
  const triggerFocused = await page.evaluate(
    () => document.activeElement?.getAttribute('data-testid')
  );
  expect(triggerFocused).toBe('open-modal');
});
```
