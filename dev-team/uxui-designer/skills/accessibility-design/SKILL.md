---
name: Accessibility Design
description: This skill should be used when the user asks about "WCAG compliance", "accessibility audit", "ARIA pattern", "screen reader", "keyboard navigation", "color contrast", "focus management", "inclusive design", or "a11y". It covers WCAG 2.2 AA/AAA compliance, ARIA authoring practices, keyboard interaction patterns, screen reader optimization, and inclusive design methodology.
---

### WCAG 2.2 AA Compliance Checklist

Accessibility compliance is organized under four principles: Perceivable, Operable, Understandable, Maintainable (POUR).

#### Perceivable

| Criterion | Requirement | How to Verify |
|-----------|-------------|---------------|
| 1.1.1 Non-text Content | All images have meaningful `alt` text or `alt=""` for decorative | Inspect every `<img>`, `<svg>`, `role="img"` |
| 1.3.1 Info and Relationships | Structure conveyed visually is also in markup (headings, lists, tables) | Check heading hierarchy, `<table>` with `<th>`, form labels |
| 1.3.5 Identify Input Purpose | Inputs have `autocomplete` attributes for personal data fields | Check name, email, address, phone, credit card fields |
| 1.4.1 Use of Color | Color is not the sole means of conveying information | Check error states, status indicators, links in text |
| 1.4.3 Contrast (Minimum) | Text: 4.5:1, Large text (18px+ bold, 24px+): 3:1 | Use contrast checker on every text/background pair |
| 1.4.11 Non-text Contrast | UI components and graphical objects: 3:1 against adjacent colors | Check borders, icons, focus indicators, chart elements |

#### Operable

| Criterion | Requirement | How to Verify |
|-----------|-------------|---------------|
| 2.1.1 Keyboard | All functionality available via keyboard | Tab through entire page, use Enter/Space/Escape/Arrows |
| 2.1.2 No Keyboard Trap | Focus can always leave a component | Test Tab and Escape from every interactive element |
| 2.4.3 Focus Order | Tab order matches visual reading order | Tab through page and verify logical sequence |
| 2.4.7 Focus Visible | Keyboard focus indicator is always visible | Tab through page and check focus rings |
| 2.4.11 Focus Not Obscured | Focused element is not fully hidden by sticky headers/footers | Scroll and tab to verify focused elements are visible |
| 2.5.8 Target Size | Interactive targets minimum 24x24px (44x44px recommended) | Measure click/tap targets |

#### Understandable

| Criterion | Requirement | How to Verify |
|-----------|-------------|---------------|
| 3.1.1 Language of Page | `<html lang="xx">` set correctly | Inspect HTML element |
| 3.2.1 On Focus | Focus does not trigger unexpected context changes | Tab to each element — nothing should auto-submit or navigate |
| 3.3.1 Error Identification | Errors are identified and described in text | Trigger all validation errors and check messages |
| 3.3.2 Labels or Instructions | Form inputs have visible labels | Check every form field has a `<label>` or `aria-label` |
| 3.3.8 Redundant Entry | Don't ask for same information twice in a process | Review multi-step forms for repeated fields |

### ARIA Authoring Patterns

Apply ARIA roles and properties for common interactive patterns. Use native HTML elements when possible — ARIA is a supplement, not a replacement.

**Rule of ARIA**: No ARIA is better than bad ARIA. Use native `<button>`, `<input>`, `<select>`, `<dialog>` before reaching for ARIA roles.

#### Dialog (Modal)

```html
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Confirm deletion</h2>
  <p>This action cannot be undone.</p>
  <button>Cancel</button>
  <button>Delete</button>
</div>
```

Keyboard behavior:
- `Tab` / `Shift+Tab`: Cycle focus within dialog (focus trap)
- `Escape`: Close dialog, return focus to trigger element
- Open: Move focus to first focusable element or dialog itself

#### Tabs

```html
<div role="tablist" aria-label="Settings sections">
  <button role="tab" aria-selected="true" aria-controls="panel-1" id="tab-1">General</button>
  <button role="tab" aria-selected="false" aria-controls="panel-2" id="tab-2" tabindex="-1">Security</button>
</div>
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">...</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>...</div>
```

Keyboard behavior:
- `Arrow Left/Right`: Move between tabs
- `Home/End`: First/last tab
- `Tab`: Move into tab panel content

#### Combobox (Autocomplete)

```html
<label for="city-input">City</label>
<input id="city-input" role="combobox" aria-expanded="true"
       aria-autocomplete="list" aria-controls="city-listbox"
       aria-activedescendant="city-option-2">
<ul id="city-listbox" role="listbox">
  <li id="city-option-1" role="option">Amsterdam</li>
  <li id="city-option-2" role="option" aria-selected="true">Austin</li>
</ul>
```

Keyboard behavior:
- `Arrow Down/Up`: Navigate options
- `Enter`: Select highlighted option
- `Escape`: Close listbox
- Type: Filter options

### Focus Management

Manage focus programmatically in these scenarios:

1. **Route changes (SPA)**: Move focus to page heading or main content landmark after navigation.
2. **Dynamic content insertion**: When new content appears (toast, inline alert), use `aria-live` or move focus if the content is critical.
3. **Modal open/close**: Trap focus inside modal on open, restore focus to trigger on close.
4. **Deletion**: When an item is removed from a list, move focus to the next item or the list container.
5. **Accordion/disclosure**: Keep focus on the trigger after expanding/collapsing.

Use `aria-live` regions for non-critical dynamic updates:

```html
<!-- Polite: announced after current speech completes -->
<div aria-live="polite" aria-atomic="true">
  3 results found
</div>

<!-- Assertive: interrupts current speech (use sparingly) -->
<div aria-live="assertive" role="alert">
  Session expiring in 2 minutes
</div>
```

### Color Contrast Requirements

| Element | Minimum Ratio (AA) | Enhanced Ratio (AAA) |
|---------|--------------------|-----------------------|
| Normal text (<18px or <14px bold) | 4.5:1 | 7:1 |
| Large text (>=18px or >=14px bold) | 3:1 | 4.5:1 |
| UI components (borders, icons) | 3:1 | Not defined |
| Focus indicators | 3:1 against adjacent colors | — |

Design techniques to meet contrast requirements:
- Use semi-transparent overlays on images behind text.
- Ensure placeholder text meets 4.5:1 (many designs fail this).
- Check contrast of disabled elements — while not required by WCAG, visible disabled states aid usability.
- Test focus indicators against ALL backgrounds the element appears on.

### Inclusive Design Methodology

Design inclusively by considering the full spectrum of user abilities:

1. **Permanent, temporary, and situational** — A user may be blind (permanent), have dilated pupils (temporary), or be in bright sunlight (situational). Design for all three.
2. **One-handed use** — Place critical interactions within thumb reach zones on mobile.
3. **Cognitive load** — Limit choices, use progressive disclosure, write at an 8th-grade reading level.
4. **Motor precision** — Large touch targets (44px minimum), generous click areas, avoid precision-dependent interactions.
5. **Slow connections** — Design loading and error states. Content should be usable before all assets load.

## References

- [aria-patterns.md](references/aria-patterns.md) - Complete ARIA pattern implementations for all common interactive widgets including menu, tree, grid, toolbar, carousel, and feed patterns with keyboard specifications.
- [audit-methodology.md](references/audit-methodology.md) - Systematic accessibility audit process, automated testing tools, manual testing procedures, screen reader testing scripts, and remediation prioritization frameworks.
