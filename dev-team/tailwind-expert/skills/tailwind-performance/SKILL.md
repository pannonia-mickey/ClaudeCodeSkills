---
name: Tailwind Performance
description: This skill should be used when the user asks about "Tailwind bundle size", "Tailwind purge", "CSS file too large", "Tailwind content config", "dynamic classes Tailwind", "Tailwind JIT", "Tailwind tree shaking", "Tailwind safelist", "CSS optimization", or "@layer Tailwind". It covers Tailwind CSS performance optimization including content scanning mechanics, dynamic class pitfalls, safe class construction patterns, JIT engine behavior, bundle size monitoring, @apply impact on CSS size, and CSS layer organization.
---

# Tailwind Performance

Optimizing Tailwind CSS output requires understanding how the framework detects class usage, generates CSS, and integrates into the build pipeline. This skill covers the mechanics of content scanning, safe patterns for dynamic styling, engine behavior across versions, and strategies for keeping bundle size under control.

## How Content Scanning Works

Tailwind scans files matching the `content` glob patterns in the configuration for complete class strings. The scanner uses regex-based extraction, not runtime analysis. It reads raw file contents and looks for complete, unbroken strings that match known utility patterns.

Key behaviors to understand:

- `bg-red-500` as a standalone string in a file is detected. `bg-${color}-500` constructed via interpolation is not.
- Any file containing Tailwind class strings must be included in the content paths. This includes JSX/TSX components, HTML templates, Markdown files with inline classes, and even JavaScript constants that hold class strings.
- The scanner does not execute JavaScript. It does not resolve variables, evaluate expressions, or follow imports. It performs a flat text scan across every file matched by the configured globs.
- Strings inside comments, disabled code blocks, and unused variables are still detected. The scanner is intentionally over-inclusive to avoid missing classes that might be used at runtime.

Configure content paths in `tailwind.config.js` (v3) or `@config` (v4) to cover every file that references Tailwind utilities.

## Dynamic Class Pitfalls

To avoid breaking Tailwind's content scanner, never construct class names using template literals, string concatenation, or dynamic segments. The scanner cannot resolve runtime values.

```typescript
// BROKEN — scanner cannot detect these
const color = 'red';
<div className={`bg-${color}-500`} />
<div className={`text-${size}`} />

// WORKS — complete strings appear in source
const colorMap = {
  red: 'bg-red-500',
  blue: 'bg-blue-500',
  green: 'bg-green-500',
} as const;
<div className={colorMap[color]} />
```

Common violations include:

- Template literal interpolation: `` `p-${spacing}` ``
- String concatenation: `'bg-' + color + '-500'`
- Array index lookups where the array is built dynamically
- Ternary expressions that compute partial class names

To fix these, always ensure the complete class string exists as a literal somewhere in the source code that the scanner can read.

## Safe Class Construction Patterns

### Lookup Objects

Map dynamic values to complete class strings. This is the simplest and most reliable pattern.

```typescript
const sizeClasses = {
  sm: 'text-sm px-2 py-1',
  md: 'text-base px-4 py-2',
  lg: 'text-lg px-6 py-3',
} as const;

<button className={sizeClasses[size]}>Click</button>
```

### CVA (Class Variance Authority)

Use CVA for type-safe variant-to-class mapping in component libraries.

```typescript
import { cva, type VariantProps } from 'class-variance-authority';

const badge = cva('inline-flex items-center rounded-full px-2 py-1 text-xs font-medium', {
  variants: {
    color: {
      gray: 'bg-gray-100 text-gray-700',
      red: 'bg-red-100 text-red-700',
      green: 'bg-green-100 text-green-700',
    },
  },
});

type BadgeProps = VariantProps<typeof badge>;
```

Every class string is written as a complete literal inside the CVA definition, making it fully scannable.

### clsx / cn Helper

Use conditional class joining with `clsx` and `tailwind-merge` to compose classes safely.

```typescript
import { clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

<div className={cn('p-4', isActive && 'bg-blue-500', isDisabled && 'opacity-50')} />
```

### Tailwind Merge

Tailwind Merge intelligently resolves conflicting utilities. Passing `p-4` and `p-6` together yields `p-6` instead of both appearing in the output. Use `twMerge` whenever component props may override base styles to prevent class conflicts.

## Safelist Configuration

To force-include classes that genuinely cannot appear as complete strings in source files, add them to the safelist. This is common when class names arrive from a CMS, database, or external API.

```javascript
// v3 tailwind.config.js
safelist: [
  'bg-red-500',
  'bg-blue-500',
  { pattern: /^bg-(red|blue|green)-(100|500|900)$/ },
]
```

Use safelist as a last resort. Prefer restructuring code to use complete class strings via lookup objects or CVA. Every safelisted class is included in the output regardless of whether it is actually used, which inflates bundle size.

## JIT Engine (v3) and Oxide Engine (v4)

- **v3 JIT**: Generates utilities on-demand as classes are detected in source files. Replaced the previous AOT (ahead-of-time) approach that generated every possible utility and then purged unused ones.
- **v4 Oxide**: Rewritten in Rust, delivering 5-10x faster builds compared to the v3 JIT engine. Uses an incremental architecture that rebuilds only what changed.
- Both engines produce only the CSS that is actually used in the scanned source files.
- In development mode, expect incremental rebuilds measured in low milliseconds. In production mode, a full scan runs against all content paths to produce the final output.

To ensure both engines work correctly, maintain clean content paths and avoid dynamic class construction.

## @apply and CSS Size Impact

Each `@apply` directive inlines the referenced utility's CSS declarations into the rule where it appears. Using it extensively duplicates CSS across multiple selectors.

- Writing `@apply flex items-center` inside 20 different selectors produces 20 copies of the same `display: flex` and `align-items: center` declarations in the output CSS.
- Prefer creating framework-level components (React, Vue, Svelte) that apply classes directly in markup, rather than abstracting them behind `@apply`.
- Limit `@apply` usage to the base layer for global element styles and to genuinely cross-file reusable patterns where a component abstraction is not feasible.

To audit `@apply` usage, search the codebase for `@apply` and evaluate whether each instance could be replaced by a shared component.

## CSS Layer Organization

Organize custom CSS using Tailwind's layer system to control specificity and output order.

```css
@layer base {
  /* Global resets, default element styles */
  h1 { @apply text-2xl font-bold; }
}

@layer components {
  /* Reusable component classes */
  .card { @apply rounded-lg border bg-white p-6 shadow-sm; }
}

@layer utilities {
  /* Custom one-off utilities */
  .text-balance { text-wrap: balance; }
}
```

- `@layer utilities` has the highest specificity in Tailwind's output order, allowing utilities to override component and base styles.
- `@layer base` has the lowest specificity, making it suitable for default styles that should be easily overridden.
- Custom styles placed outside of any `@layer` directive override all layered styles, which can cause unexpected specificity conflicts. Keep all custom CSS inside an appropriate layer.

## Bundle Size Monitoring

Target production CSS output under 30KB gzipped for most projects. Larger applications with extensive design systems may exceed this, but treat it as a budget to track against.

To measure the current CSS output size:

```bash
npx tailwindcss -o output.css --minify && gzip -c output.css | wc -c
```

To integrate into CI, add a step that builds the CSS, measures the gzipped size, and fails the build if it exceeds a defined threshold. This prevents gradual bundle growth from going unnoticed.

Tools for monitoring:

- `bundlephobia` for JavaScript dependency size tracking
- Manual gzip measurement for CSS (shown above)
- Webpack Bundle Analyzer or Vite's `rollup-plugin-visualizer` for overall asset inspection

Review safelist entries periodically and remove any that are no longer needed. Audit content paths to ensure they are not scanning irrelevant files that inflate the detected class set.

## References

- [Build Optimization](references/build-optimization.md) — deep-dive into PostCSS pipeline, CSS minification, critical CSS extraction, and production build checklist.
