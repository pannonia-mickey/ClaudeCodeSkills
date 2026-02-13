---
name: UX Design Mastery
description: This skill should be used when the user asks about "design system", "design tokens", "component specification", "visual hierarchy", "typography scale", "color system", "spacing system", "UI specification", or "design-to-code". It covers design system architecture, token hierarchies, component anatomy, visual design principles, and production-ready UI specifications.
---

### Design Token Architecture

Design tokens are the atomic values of a design system. Organize them in three tiers for maximum scalability.

#### Tier 1: Primitive Tokens (Raw Values)

Primitive tokens define the complete palette of available values with no semantic meaning.

```css
/* Color primitives */
--color-blue-50: #eff6ff;
--color-blue-100: #dbeafe;
--color-blue-500: #3b82f6;
--color-blue-600: #2563eb;
--color-blue-700: #1d4ed8;
--color-blue-900: #1e3a5a;

/* Spacing primitives (4px base grid) */
--space-1: 0.25rem;   /* 4px */
--space-2: 0.5rem;    /* 8px */
--space-3: 0.75rem;   /* 12px */
--space-4: 1rem;      /* 16px */
--space-6: 1.5rem;    /* 24px */
--space-8: 2rem;      /* 32px */
--space-12: 3rem;     /* 48px */
--space-16: 4rem;     /* 64px */

/* Typography primitives */
--font-size-xs: 0.75rem;    /* 12px */
--font-size-sm: 0.875rem;   /* 14px */
--font-size-base: 1rem;     /* 16px */
--font-size-lg: 1.125rem;   /* 18px */
--font-size-xl: 1.25rem;    /* 20px */
--font-size-2xl: 1.5rem;    /* 24px */
--font-size-3xl: 1.875rem;  /* 30px */
--font-size-4xl: 2.25rem;   /* 36px */
```

#### Tier 2: Semantic Tokens (Purpose-Driven)

Semantic tokens map primitives to purposes. These are what components consume.

```css
/* Semantic color tokens */
--color-text-primary: var(--color-gray-900);
--color-text-secondary: var(--color-gray-600);
--color-text-disabled: var(--color-gray-400);
--color-text-inverse: var(--color-white);
--color-text-link: var(--color-blue-600);
--color-text-error: var(--color-red-600);
--color-text-success: var(--color-green-600);

--color-bg-primary: var(--color-white);
--color-bg-secondary: var(--color-gray-50);
--color-bg-elevated: var(--color-white);
--color-bg-overlay: rgba(0, 0, 0, 0.5);

--color-border-default: var(--color-gray-200);
--color-border-focus: var(--color-blue-500);
--color-border-error: var(--color-red-500);

/* Semantic spacing */
--space-component-gap: var(--space-3);
--space-section-gap: var(--space-8);
--space-page-gutter: var(--space-4);
--space-input-padding-x: var(--space-3);
--space-input-padding-y: var(--space-2);
```

#### Tier 3: Component Tokens (Scoped)

Component tokens scope semantic tokens to specific components, enabling targeted overrides.

```css
/* Button component tokens */
--button-padding-x: var(--space-4);
--button-padding-y: var(--space-2);
--button-font-size: var(--font-size-sm);
--button-font-weight: var(--font-weight-medium);
--button-border-radius: var(--radius-md);
--button-primary-bg: var(--color-bg-accent);
--button-primary-text: var(--color-text-inverse);
--button-primary-hover-bg: var(--color-bg-accent-hover);
```

### Typography Scale

Build typography scales using a modular ratio for mathematical harmony.

| Name | Size | Line Height | Weight | Use |
|------|------|-------------|--------|-----|
| `display-lg` | 3rem (48px) | 1.1 | 700 | Hero headlines |
| `display-sm` | 2.25rem (36px) | 1.2 | 700 | Page titles |
| `heading-lg` | 1.5rem (24px) | 1.3 | 600 | Section headings |
| `heading-sm` | 1.125rem (18px) | 1.4 | 600 | Card headers |
| `body-lg` | 1rem (16px) | 1.5 | 400 | Primary body text |
| `body-sm` | 0.875rem (14px) | 1.5 | 400 | Secondary text, labels |
| `caption` | 0.75rem (12px) | 1.4 | 400 | Helper text, metadata |

Apply these rules:
- Body text minimum 16px on mobile for readability.
- Line height decreases as font size increases (large headings need tighter leading).
- Limit to 2-3 font weights per project to maintain visual consistency.
- Use `letter-spacing: -0.02em` for headings above 24px to tighten tracking.

### Component Specification Format

To specify a component, document all facets that an engineer needs for implementation.

**Anatomy**: Name every sub-element (root, label, input, helper-text, error-message, icon-leading, icon-trailing).

**States**: Document every visual state with exact token changes:
- Default, Hover, Focus-visible, Active, Disabled, Read-only
- For inputs: Empty, Filled, Error, Success
- For async: Loading, Loaded, Error, Empty

**Spacing**: All internal and external spacing using token references, never raw pixel values.

**Responsive behavior**: How the component adapts across breakpoints (full-width on mobile, fixed-width on desktop, stacking behavior).

**Keyboard interaction**: Tab order, keyboard shortcuts, focus trap behavior if modal.

**Accessibility**: ARIA roles, labels, descriptions, live regions for dynamic content.

### Visual Hierarchy Principles

Establish clear scanning patterns using these tools in combination:

1. **Size contrast** — Primary content at least 1.5x the size of secondary content.
2. **Weight contrast** — Semibold/bold for headings, regular for body. Never use more than 3 weights.
3. **Color contrast** — Primary text in high-contrast color, secondary text reduced (but still meeting 4.5:1 ratio).
4. **Spatial grouping** — Related items closer together (Gestalt proximity). Space between groups > space within groups.
5. **Alignment** — Left-align content for LTR reading flow. Limit alignment axes to reduce visual noise.

### Color System Design

Build color systems with accessibility baked in:

1. **Generate a 10-step scale** for each hue (50-900) ensuring consistent perceived lightness steps.
2. **Map semantic roles**: primary, secondary, success, warning, error, info, neutral.
3. **Verify contrast pairs**: every foreground/background combination meets WCAG AA (4.5:1 text, 3:1 UI).
4. **Design dark mode** by remapping semantic tokens to dark primitives — not by inverting.
5. **Test with color blindness simulators** (protanopia, deuteranopia, tritanopia) — never rely on color alone.

## References

- [solid-patterns.md](references/solid-patterns.md) - UX/UI-specific SOLID principles applied to design system architecture, component API design, and design token organization.
- [best-practices.md](references/best-practices.md) - Modern CSS techniques, design system governance, design-to-code workflows, and production design specification standards.
