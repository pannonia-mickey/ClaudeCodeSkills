---
name: Responsive Design
description: This skill should be used when the user asks about "responsive layout", "mobile-first design", "breakpoints", "fluid typography", "container queries", "responsive images", "adaptive layout", or "CSS grid layout". It covers mobile-first strategy, breakpoint systems, fluid sizing techniques, container queries, responsive component patterns, and modern CSS layout approaches.
---

### Mobile-First Breakpoint Strategy

Design from smallest viewport up. Mobile styles are the default; larger viewports add complexity.

#### Standard Breakpoint Scale

```css
/* Mobile-first breakpoints */
--breakpoint-sm: 640px;   /* Large phones, small tablets */
--breakpoint-md: 768px;   /* Tablets portrait */
--breakpoint-lg: 1024px;  /* Tablets landscape, small laptops */
--breakpoint-xl: 1280px;  /* Laptops, desktops */
--breakpoint-2xl: 1536px; /* Large desktops */
```

```css
/* Mobile-first media queries (always use min-width) */
.card-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--space-4);
}

@media (min-width: 640px) {
  .card-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1024px) {
  .card-grid {
    grid-template-columns: repeat(3, 1fr);
    gap: var(--space-6);
  }
}

@media (min-width: 1280px) {
  .card-grid {
    grid-template-columns: repeat(4, 1fr);
  }
}
```

Rules for breakpoint usage:
- Never design for specific devices. Design for content needs.
- Add breakpoints when the layout breaks, not at predetermined device widths.
- Prefer fewer breakpoints. Most layouts need only 2-3 transitions.
- Use the standard scale as starting points, not rigid rules.

### Fluid Typography

Use `clamp()` for typography that scales smoothly between viewport sizes without breakpoint jumps.

```css
/* Formula: clamp(min, preferred, max) */
/* preferred uses viewport units for fluid scaling */

--font-size-body: clamp(1rem, 0.95rem + 0.25vw, 1.125rem);
--font-size-h1: clamp(2rem, 1.5rem + 2.5vw, 3.5rem);
--font-size-h2: clamp(1.5rem, 1.25rem + 1.25vw, 2.25rem);
--font-size-h3: clamp(1.25rem, 1.1rem + 0.75vw, 1.75rem);
```

Apply fluid typography rules:
- Minimum font size for body text: 16px (1rem). Never go below.
- Maximum heading size: cap at a readable maximum to prevent absurdly large text on ultrawide displays.
- Line length: Maintain 45-75 characters per line using `max-width: 65ch` on text containers.
- Line height: Use unitless values (`line-height: 1.5`) so it scales with font size.

### Container Queries

Container queries enable components to respond to their container's size rather than the viewport. This makes components truly reusable across different layout contexts.

```css
/* Define a containment context */
.card-container {
  container-type: inline-size;
  container-name: card;
}

/* Query the container's width */
@container card (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 200px 1fr;
    gap: var(--space-4);
  }
}

@container card (min-width: 600px) {
  .card {
    grid-template-columns: 250px 1fr auto;
  }

  .card__actions {
    flex-direction: column;
  }
}
```

When to use container queries vs media queries:
- **Container queries**: Component layout depends on available space (cards, widgets, sidebar content).
- **Media queries**: Page-level layout changes (navigation collapse, grid columns, page gutter).

### Responsive Layout Patterns

#### Sidebar + Main Content

```css
.layout {
  display: grid;
  grid-template-columns: 1fr;
  min-height: 100dvh;
}

@media (min-width: 1024px) {
  .layout {
    grid-template-columns: 280px 1fr;
  }
}
```

On mobile, the sidebar becomes a slide-out drawer or bottom sheet. On desktop, it becomes a fixed sidebar.

#### Holy Grail (Header + Sidebar + Main + Footer)

```css
.page {
  display: grid;
  grid-template-rows: auto 1fr auto;
  grid-template-columns: 1fr;
  min-height: 100dvh;
}

@media (min-width: 1024px) {
  .page {
    grid-template-columns: 240px 1fr;
    grid-template-rows: auto 1fr auto;
  }

  .header { grid-column: 1 / -1; }
  .footer { grid-column: 1 / -1; }
}
```

#### Card Grid with Auto-Fill

```css
/* Self-responsive grid — no media queries needed */
.auto-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(min(280px, 100%), 1fr));
  gap: var(--space-4);
}
```

The `min(280px, 100%)` pattern prevents overflow on narrow viewports where 280px exceeds container width.

### Responsive Images

```html
<!-- Art direction: different crops for different viewports -->
<picture>
  <source media="(min-width: 1024px)" srcset="hero-wide.webp" />
  <source media="(min-width: 640px)" srcset="hero-medium.webp" />
  <img src="hero-narrow.webp" alt="Product showcase" loading="lazy" />
</picture>

<!-- Resolution switching: same image, different sizes -->
<img
  src="product-400.webp"
  srcset="product-400.webp 400w, product-800.webp 800w, product-1200.webp 1200w"
  sizes="(min-width: 1024px) 33vw, (min-width: 640px) 50vw, 100vw"
  alt="Product name"
  loading="lazy"
  decoding="async"
/>
```

Rules:
- Always provide `width` and `height` attributes to prevent layout shift.
- Use `loading="lazy"` for below-the-fold images.
- Use WebP/AVIF formats with JPEG/PNG fallbacks.
- Set `sizes` attribute to match actual rendered size, not just viewport fractions.

### Responsive Component Behavior

Document how components adapt across breakpoints:

| Component | Mobile (<640px) | Tablet (640-1023px) | Desktop (>=1024px) |
|-----------|----------------|--------------------|--------------------|
| Navigation | Bottom tab bar or hamburger drawer | Top bar with horizontal links | Top bar with full menu + search |
| Data table | Card list (stacked) | Scrollable table | Full table with all columns |
| Modal | Full-screen sheet | Centered dialog (max-width: 560px) | Centered dialog (max-width: 640px) |
| Sidebar | Hidden, toggle via button | Collapsible panel | Persistent sidebar |
| Form | Single column, full-width inputs | Two-column for short fields | Two or three columns with section grouping |

### Modern CSS Techniques

Prefer modern CSS features that reduce the need for breakpoints:

```css
/* Logical properties for RTL/LTR support */
.card {
  padding-inline: var(--space-4);    /* left/right in LTR, right/left in RTL */
  padding-block: var(--space-3);     /* top/bottom */
  margin-inline-start: var(--space-2); /* left in LTR, right in RTL */
}

/* Dynamic viewport units */
.hero {
  min-height: 100dvh; /* accounts for mobile browser chrome */
}

/* Aspect ratio for responsive media */
.video-embed {
  aspect-ratio: 16 / 9;
  width: 100%;
}

/* Subgrid for aligned nested layouts */
.card-grid > .card {
  display: grid;
  grid-template-rows: subgrid;
  grid-row: span 3; /* header, body, footer aligned across cards */
}
```

## References

- [layout-patterns.md](references/layout-patterns.md) - Comprehensive responsive layout patterns including dashboard layouts, split-view patterns, masonry grids, responsive navigation patterns, and adaptive content strategies.
- [css-techniques.md](references/css-techniques.md) - Modern CSS capabilities for responsive design including cascade layers, :has() selector, scroll-driven animations, view transitions, anchor positioning, and CSS nesting.
