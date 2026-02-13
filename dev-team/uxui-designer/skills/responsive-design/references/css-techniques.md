# Modern CSS Techniques Reference

## Cascade Layers (@layer)

Cascade layers provide explicit control over specificity ordering, solving the "specificity wars" problem in large design systems.

```css
/* Declare layer order — first declared = lowest priority */
@layer reset, tokens, base, components, utilities, overrides;

/* Reset layer — lowest priority */
@layer reset {
  *, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
}

/* Component layer */
@layer components {
  .btn {
    padding: var(--space-2) var(--space-4);
    border-radius: var(--radius-md);
  }
}

/* Utility layer — always wins over components */
@layer utilities {
  .mt-4 { margin-block-start: var(--space-4) !important; }
  .hidden { display: none !important; }
}

/* Third-party styles in their own layer */
@layer external {
  @import url('third-party-lib.css');
}
```

**Key benefit:** Utilities always override component styles regardless of selector specificity, because the `utilities` layer is declared after `components`.

---

## :has() Selector

The `:has()` relational pseudo-class enables parent selection and context-aware styling.

### Context-Aware Component Styling

```css
/* Card layout changes when it contains an image */
.card:has(> .card__image) {
  display: grid;
  grid-template-columns: 200px 1fr;
}

/* Form group shows error state when input is invalid */
.form-group:has(input:user-invalid) {
  --field-border: var(--color-border-error);
}

.form-group:has(input:user-invalid) .form-group__label {
  color: var(--color-text-error);
}

/* Navigation highlights parent when child link is current */
.nav-item:has(> a[aria-current="page"]) {
  background: var(--color-bg-active);
  border-inline-start: 3px solid var(--color-primary);
}

/* Body class-free dark mode */
html:has(#dark-mode-toggle:checked) {
  color-scheme: dark;
  --color-bg-primary: var(--color-gray-900);
  --color-text-primary: var(--color-gray-50);
}
```

### Responsive Behavior Without JS

```css
/* Show/hide sidebar based on checkbox toggle */
.layout:has(#sidebar-toggle:checked) .sidebar {
  transform: translateX(0);
}

/* Adjust grid when sidebar is visible */
.layout:has(#sidebar-toggle:checked) .main {
  margin-inline-start: var(--sidebar-width);
}

/* Empty state when list has no visible items */
.list:has(> :not([hidden])):not(:has(> :not([hidden]) ~ :not([hidden]))) {
  /* Only one visible item */
}

.list:not(:has(> :not([hidden]))) .empty-state {
  display: block;
}
```

---

## Scroll-Driven Animations

### Scroll Progress Indicator

```css
.progress-bar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 3px;
  background: var(--color-primary);
  transform-origin: left;
  animation: fill-bar linear both;
  animation-timeline: scroll(root block);
}

@keyframes fill-bar {
  from { transform: scaleX(0); }
  to { transform: scaleX(1); }
}
```

### Element Reveal on Scroll

```css
.reveal-on-scroll {
  animation: reveal linear both;
  animation-timeline: view();
  animation-range: entry 10% entry 90%;
}

@keyframes reveal {
  from {
    opacity: 0;
    transform: translateY(40px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

### Sticky Header Shadow on Scroll

```css
.sticky-header {
  position: sticky;
  top: 0;
  animation: shadow-on-scroll linear both;
  animation-timeline: scroll(nearest block);
  animation-range: 0px 100px;
}

@keyframes shadow-on-scroll {
  from { box-shadow: none; }
  to { box-shadow: var(--shadow-md); }
}
```

---

## View Transitions API

### SPA Page Transitions

```css
/* Default crossfade transition */
::view-transition-old(root) {
  animation: 200ms ease-out fade-out;
}

::view-transition-new(root) {
  animation: 300ms ease-out fade-in;
}

@keyframes fade-out {
  to { opacity: 0; }
}

@keyframes fade-in {
  from { opacity: 0; }
}
```

### Shared Element Transitions

```css
/* Assign transition names to matching elements */
.card-thumbnail {
  view-transition-name: hero-image;
}

.detail-hero {
  view-transition-name: hero-image;
}

/* Customize the shared element animation */
::view-transition-group(hero-image) {
  animation-duration: 400ms;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}
```

```javascript
// Trigger transition in SPA router
async function navigateTo(url) {
  if (!document.startViewTransition) {
    updateDOM(url);
    return;
  }

  const transition = document.startViewTransition(() => updateDOM(url));
  await transition.finished;
}
```

---

## Anchor Positioning

Position elements relative to other elements without JavaScript.

```css
/* Define an anchor */
.trigger-button {
  anchor-name: --menu-anchor;
}

/* Position relative to anchor */
.dropdown-menu {
  position: fixed;
  position-anchor: --menu-anchor;

  /* Place below the anchor, aligned to its start edge */
  inset-area: block-end span-inline-end;

  /* Fallback if not enough space below */
  position-try-fallbacks:
    --above,      /* Try above */
    --left;       /* Try to the left */

  /* Auto-sizing */
  max-height: calc(100dvh - anchor(bottom) - var(--space-2));
}

@position-try --above {
  inset-area: block-start span-inline-end;
}

@position-try --left {
  inset-area: inline-start span-block-end;
}
```

---

## CSS Nesting

Native CSS nesting for maintainable component styles.

```css
.card {
  background: var(--color-bg-elevated);
  border-radius: var(--radius-lg);
  overflow: hidden;

  /* Direct child selectors */
  & .card__header {
    padding: var(--space-4);
    border-block-end: 1px solid var(--color-border-default);
  }

  & .card__body {
    padding: var(--space-4);
  }

  /* State modifiers */
  &.card--highlighted {
    border: 2px solid var(--color-primary);
  }

  /* Pseudo-classes */
  &:hover {
    box-shadow: var(--shadow-md);
  }

  &:focus-within {
    outline: 2px solid var(--color-border-focus);
    outline-offset: 2px;
  }

  /* Media queries nest inside components */
  @media (min-width: 768px) {
    display: grid;
    grid-template-columns: 200px 1fr;
  }

  /* Container queries too */
  @container (min-width: 400px) {
    display: grid;
    grid-template-columns: 200px 1fr;
  }
}
```

---

## @scope

Scope styles to a subtree without relying on class naming conventions.

```css
@scope (.card) to (.card__footer) {
  /* Styles apply inside .card but NOT inside .card__footer */
  a {
    color: var(--color-text-link);
    text-decoration: underline;
  }

  p {
    color: var(--color-text-secondary);
    line-height: 1.6;
  }
}

/* Component isolation */
@scope (.sidebar) {
  nav {
    display: flex;
    flex-direction: column;
    gap: var(--space-1);
  }

  a {
    padding: var(--space-2) var(--space-3);
    border-radius: var(--radius-sm);
    color: var(--color-text-primary);
    text-decoration: none;

    &:hover {
      background: var(--color-bg-hover);
    }

    &[aria-current="page"] {
      background: var(--color-bg-active);
      font-weight: var(--font-weight-semibold);
    }
  }
}
```

---

## color-mix() and Relative Color Syntax

### Dynamic Color Variations

```css
:root {
  --color-primary: oklch(0.55 0.23 255);

  /* Mix with white for lighter tints */
  --color-primary-light: color-mix(in oklch, var(--color-primary) 30%, white);

  /* Mix with black for darker shades */
  --color-primary-dark: color-mix(in oklch, var(--color-primary) 70%, black);

  /* Semi-transparent variant */
  --color-primary-alpha-20: color-mix(in oklch, var(--color-primary) 20%, transparent);
}

/* Hover states via color-mix */
.btn-primary {
  background: var(--color-primary);
}

.btn-primary:hover {
  background: color-mix(in oklch, var(--color-primary) 85%, black);
}
```

### Relative Color Syntax

```css
:root {
  --brand: oklch(0.6 0.2 250);

  /* Lighten by increasing lightness */
  --brand-light: oklch(from var(--brand) calc(l + 0.2) c h);

  /* Darken by decreasing lightness */
  --brand-dark: oklch(from var(--brand) calc(l - 0.15) c h);

  /* Desaturate by reducing chroma */
  --brand-muted: oklch(from var(--brand) l calc(c * 0.5) h);

  /* Shift hue for complementary color */
  --brand-complement: oklch(from var(--brand) l c calc(h + 180));

  /* Add transparency */
  --brand-transparent: oklch(from var(--brand) l c h / 0.5);
}
```

---

## Container Style Queries

Query computed style values of a container, not just its size.

```css
.card-container {
  container-name: card;
  container-type: normal; /* style queries don't need size containment */
}

/* Theme-aware component */
@container card style(--theme: dark) {
  .card {
    background: var(--color-gray-800);
    color: var(--color-gray-100);
  }
}

@container card style(--density: compact) {
  .card {
    padding: var(--space-2);
    font-size: var(--font-size-sm);
  }
}
```

---

## Popover API

Native popover behavior without JavaScript for tooltips, menus, and dialogs.

```html
<button popovertarget="my-popover">Open Menu</button>

<div id="my-popover" popover>
  <nav>
    <a href="/settings">Settings</a>
    <a href="/profile">Profile</a>
    <a href="/logout">Log out</a>
  </nav>
</div>
```

```css
[popover] {
  margin: 0;
  padding: var(--space-3);
  border: 1px solid var(--color-border-default);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-lg);
  background: var(--color-bg-elevated);

  /* Entry/exit animations */
  opacity: 0;
  transform: translateY(-8px);
  transition: opacity var(--duration-fast) var(--ease-out),
              transform var(--duration-fast) var(--ease-out),
              overlay var(--duration-fast) allow-discrete,
              display var(--duration-fast) allow-discrete;

  &:popover-open {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Starting style for entry animation */
@starting-style {
  [popover]:popover-open {
    opacity: 0;
    transform: translateY(-8px);
  }
}

/* Backdrop */
[popover]::backdrop {
  background: rgba(0, 0, 0, 0.3);
}
```

---

## interpolate-size

Transition to and from `auto` dimensions — previously impossible in CSS.

```css
/* Enable interpolation to/from auto */
:root {
  interpolate-size: allow-keywords;
}

/* Collapsible panel with height transition */
.panel__content {
  height: 0;
  overflow: hidden;
  transition: height var(--duration-normal) var(--ease-default);
}

.panel--open .panel__content {
  height: auto; /* Transitions smoothly from 0 to auto */
}

/* Expanding search input */
.search-input {
  width: 40px;
  transition: width var(--duration-normal) var(--ease-default);
}

.search-input:focus {
  width: auto; /* Transitions smoothly to content width */
}
```

---

## Field-Sizing

Auto-growing inputs and textareas without JavaScript.

```css
textarea {
  field-sizing: content;
  min-height: 3lh; /* Minimum 3 lines */
  max-height: 10lh; /* Maximum 10 lines */
}

select {
  field-sizing: content; /* Width matches longest option */
}

input {
  field-sizing: content;
  min-width: 10ch; /* Minimum 10 characters wide */
}
```
