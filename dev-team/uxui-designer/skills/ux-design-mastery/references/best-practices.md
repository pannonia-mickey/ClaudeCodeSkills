# Design System Best Practices

## Modern CSS for Design Systems

### CSS Custom Properties as Token Layer

```css
/* Primitive layer — raw values */
@layer tokens.primitives {
  :root {
    --color-blue-500: oklch(0.62 0.21 255);
    --color-blue-600: oklch(0.55 0.23 255);
    --font-size-base: 1rem;
    --space-unit: 0.25rem;
  }
}

/* Semantic layer — purpose-driven mapping */
@layer tokens.semantic {
  :root {
    --color-primary: var(--color-blue-600);
    --color-primary-hover: var(--color-blue-700);
    --text-body: var(--font-size-base);
    --space-4: calc(var(--space-unit) * 4);
  }
}

/* Component layer — scoped overrides */
@layer tokens.components {
  :root {
    --btn-bg: var(--color-primary);
    --btn-padding: var(--space-2) var(--space-4);
  }
}
```

### Cascade Layers for Specificity Control

```css
/* Define layer order — lower layers are overridden by higher */
@layer reset, tokens, base, components, utilities, overrides;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; margin: 0; }
}

@layer base {
  body { font-family: var(--font-family-body); color: var(--color-text-primary); }
}

@layer components {
  .btn { /* component styles */ }
  .card { /* component styles */ }
}

@layer utilities {
  .sr-only { /* screen reader only */ }
  .truncate { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
}
```

### :has() for Context-Aware Components

```css
/* Card gets a different layout when it has an image */
.card:has(.card__image) {
  display: grid;
  grid-template-columns: 200px 1fr;
}

/* Form group shows error style when its input is invalid */
.form-group:has(input:invalid:not(:placeholder-shown)) {
  --field-border: var(--color-border-error);
  --field-label-color: var(--color-text-error);
}

/* Navigation item is highlighted when it contains the current page link */
.nav-item:has(a[aria-current="page"]) {
  background: var(--color-bg-active);
}
```

### CSS Nesting for Component Authoring

```css
.card {
  background: var(--color-bg-elevated);
  border-radius: var(--radius-lg);
  padding: var(--space-4);

  & .card__title {
    font-size: var(--font-size-lg);
    font-weight: var(--font-weight-semibold);
  }

  & .card__body {
    margin-block-start: var(--space-3);
    color: var(--color-text-secondary);
  }

  &:hover {
    box-shadow: var(--shadow-md);
  }

  @container card (min-width: 400px) {
    display: grid;
    grid-template-columns: auto 1fr;
  }
}
```

---

## Design System Governance

### Versioning Strategy

Use semantic versioning for design system packages:

| Change Type | Version Bump | Example |
|-------------|-------------|---------|
| New component or token | Minor (0.x.0) | Add `<Tooltip>` component |
| Bug fix, visual tweak | Patch (0.0.x) | Fix focus ring on `<Button>` |
| Breaking API change | Major (x.0.0) | Rename `variant` prop to `appearance` |
| Token value adjustment | Patch | Change `--space-4` from 16px to 14px |
| Token removal | Major | Remove `--color-brand-accent` |

### Deprecation Process

1. **Announce** — Mark deprecated in documentation and JSDoc/TSDoc comments
2. **Warn** — Add console warnings in development builds
3. **Migrate** — Provide codemod or migration guide
4. **Remove** — Delete after 2 major versions

```tsx
/** @deprecated Use `appearance` instead. Will be removed in v4.0 */
interface ButtonProps {
  variant?: 'primary' | 'secondary';
  appearance?: 'primary' | 'secondary';
}
```

### Contribution Guidelines

A new component proposal must include:
1. **Use case** — At least 3 distinct use cases across the product
2. **API design** — Proposed props with TypeScript interface
3. **States** — All visual states documented
4. **Accessibility** — ARIA pattern, keyboard behavior, screen reader behavior
5. **Responsive behavior** — Mobile and desktop adaptations
6. **Existing alternatives** — Why existing components cannot fulfill the need

---

## Design-to-Code Workflows

### Token Export Pipeline

```
Design Tool (Figma/Tokens Studio)
    ↓ export as JSON
Token Transform (Style Dictionary)
    ↓ generate platform outputs
├── CSS custom properties (web)
├── Swift/Kotlin constants (native)
├── JSON (documentation)
└── TypeScript types (type safety)
```

### Style Dictionary Configuration

```json
{
  "source": ["tokens/**/*.json"],
  "platforms": {
    "css": {
      "transformGroup": "css",
      "buildPath": "dist/css/",
      "files": [{
        "destination": "tokens.css",
        "format": "css/variables",
        "options": { "outputReferences": true }
      }]
    },
    "ts": {
      "transformGroup": "js",
      "buildPath": "dist/ts/",
      "files": [{
        "destination": "tokens.ts",
        "format": "javascript/es6"
      }]
    }
  }
}
```

### Handoff Specification Format

Every component handoff includes:

```markdown
## Component: [Name]

### Anatomy
[Labeled diagram of sub-elements]

### Props / API
| Prop | Type | Default | Description |
|------|------|---------|-------------|

### Tokens Used
| Token | Value | Usage |
|-------|-------|-------|

### States
| State | Visual Changes | Token Overrides |
|-------|---------------|-----------------|

### Spacing
[Annotated spacing diagram with token references]

### Responsive Behavior
| Breakpoint | Layout Change |
|-----------|---------------|

### Keyboard Interaction
| Key | Action |
|-----|--------|

### ARIA
| Attribute | Value | Purpose |
|-----------|-------|---------|
```

---

## Token Naming Conventions

### Comparison of Approaches

| Approach | Example | Pros | Cons |
|----------|---------|------|------|
| T-shirt sizes | `--space-sm`, `--space-md` | Intuitive | Ambiguous mid-values |
| Numeric scale | `--space-1`, `--space-2` | Precise ordering | Not self-documenting |
| Base multiplier | `--space-4` (= 4 * base) | Mathematical, predictable | Requires knowing base unit |
| Semantic | `--space-tight`, `--space-loose` | Descriptive | Subjective interpretation |

**Recommended: Base multiplier for primitives, semantic for component tokens.**

```css
/* Primitives: numeric base multiplier (4px base) */
--space-1: 0.25rem;
--space-2: 0.5rem;
--space-4: 1rem;
--space-8: 2rem;

/* Semantic: descriptive purpose */
--space-component-gap: var(--space-3);
--space-section-gap: var(--space-8);
--space-input-padding: var(--space-2) var(--space-3);
```

### Token Naming Structure

Follow this pattern: `--{category}-{property}-{variant}-{state}`

```css
--color-bg-primary              /* category-property-variant */
--color-bg-primary-hover        /* category-property-variant-state */
--color-text-error              /* category-property-variant */
--font-size-heading-lg          /* category-property-variant-scale */
--space-inline-md               /* category-property-scale */
--shadow-elevation-high         /* category-property-scale */
```

---

## Multi-Brand / Multi-Theme Architecture

### Token Layer Strategy

```
┌─────────────────────────────┐
│  Brand A Tokens             │  Overrides semantic layer
│  (theme-brand-a.css)        │
├─────────────────────────────┤
│  Semantic Tokens            │  Shared across brands
│  (tokens-semantic.css)      │
├─────────────────────────────┤
│  Primitive Tokens           │  Complete value palette
│  (tokens-primitives.css)    │
└─────────────────────────────┘
```

### Implementation

```css
/* primitives.css — shared palette */
:root {
  --color-blue-500: oklch(0.62 0.21 255);
  --color-purple-500: oklch(0.55 0.24 300);
}

/* semantic.css — default brand mapping */
:root {
  --color-primary: var(--color-blue-500);
  --color-accent: var(--color-blue-300);
}

/* theme-brand-b.css — brand override */
[data-brand="b"] {
  --color-primary: var(--color-purple-500);
  --color-accent: var(--color-purple-300);
}

/* dark.css — darkness mode (independent of brand) */
[data-mode="dark"] {
  --color-bg-primary: var(--color-gray-900);
  --color-text-primary: var(--color-gray-50);
}
```

Brands and modes compose independently: `data-brand="b" data-mode="dark"` gives Brand B in dark mode without any additional token definitions.

---

## Component Documentation Template

Every published component in the design system should include this documentation:

```markdown
# ComponentName

## Overview
One-sentence description of what this component does and when to use it.

## Usage
\`\`\`jsx
import { ComponentName } from '@design-system/components';

<ComponentName variant="primary" size="md">
  Content
</ComponentName>
\`\`\`

## Props
| Prop | Type | Default | Required | Description |
|------|------|---------|----------|-------------|

## Variants
Visual examples of each variant with usage guidance.

## Sizes
Visual examples of each size with spacing specifications.

## States
| State | Description | Screenshot |
|-------|-------------|------------|

## Accessibility
- **Role**: [ARIA role]
- **Keyboard**: [Key interaction table]
- **Screen Reader**: [Announcement behavior]

## Do / Don't
| Do | Don't |
|----|-------|

## Related Components
- [ComponentA] — Use when [scenario]
- [ComponentB] — Use when [different scenario]
```
