---
name: Tailwind Configuration
description: This skill should be used when the user asks about "tailwind.config", "Tailwind theme", "custom colors Tailwind", "Tailwind plugin", "Tailwind preset", "design tokens Tailwind", "@theme directive", "Tailwind v4 config", "Tailwind extend", "Tailwind content paths", or "Tailwind IntelliSense". It covers Tailwind CSS configuration including theme customization, extending defaults, plugin basics, presets for shared configs, v4 CSS-first configuration with @theme, content path setup, and IDE tooling.
---

# Tailwind Configuration

This skill covers Tailwind CSS configuration from project setup through theme customization, plugin integration, and tooling. Apply these patterns to establish a scalable design system within any Tailwind-based project.

## v3 Configuration (tailwind.config.js/ts)

Structure every v3 configuration file around four top-level keys: `content`, `theme`, `plugins`, and `presets`. Place the file at the project root as `tailwind.config.js` (or `.ts` for TypeScript projects).

Use `theme.extend` to add new values while preserving all of Tailwind's defaults. Override `theme` directly only when replacing the entire default scale is intentional -- for example, restricting a project to a fixed set of brand colors with no access to Tailwind's default palette.

```javascript
import type { Config } from 'tailwindcss';

export default {
  content: ['./src/**/*.{js,ts,jsx,tsx,mdx}'],
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#f0f5ff',
          500: '#3b82f6',
          900: '#1e3a5f',
        },
      },
      fontFamily: {
        heading: ['Inter', 'sans-serif'],
        body: ['Source Sans 3', 'sans-serif'],
      },
      spacing: {
        '18': '4.5rem',
        '88': '22rem',
      },
      borderRadius: {
        '4xl': '2rem',
      },
    },
  },
  plugins: [],
} satisfies Config;
```

Note the `satisfies Config` assertion at the end. This provides full type checking and autocompletion in TypeScript without narrowing the export type, making it compatible with tooling that reads the config at runtime.

## v4 CSS-First Configuration (@theme)

Starting with Tailwind CSS v4, define all theme customizations directly in CSS using the `@theme` directive. Import Tailwind at the top of the main stylesheet and declare custom design tokens inside `@theme`:

```css
@import "tailwindcss";

@theme {
  --color-brand-50: #f0f5ff;
  --color-brand-500: #3b82f6;
  --color-brand-900: #1e3a5f;
  --font-heading: "Inter", sans-serif;
  --font-body: "Source Sans 3", sans-serif;
  --breakpoint-3xl: 120rem;
  --ease-fluid: cubic-bezier(0.3, 0, 0, 1);
}
```

Organize tokens by namespace. Each namespace maps to a set of utilities:

- `--color-*` generates color utilities like `bg-brand-500`, `text-brand-900`
- `--font-*` generates font-family utilities like `font-heading`
- `--breakpoint-*` generates responsive prefixes like `3xl:`
- `--spacing-*` generates spacing utilities for margin, padding, gap
- `--ease-*` generates easing utilities for transitions
- `--animate-*` generates animation utilities

To remove all default values from a namespace, set it to `initial`. For example, `--color-*: initial;` clears every default color, leaving only the custom colors defined afterward. This is the v4 equivalent of overriding `theme.colors` directly in v3.

Reference any theme value elsewhere in CSS with standard custom property syntax: `var(--color-brand-500)`. These variables are available globally, not just within Tailwind utility classes.

## Custom Colors

Define color scales with semantic names that communicate purpose rather than hue. Prefer names like `brand`, `accent`, `surface`, and `danger` over `blue` or `red`. Build each scale with numeric stops (50 through 950) to provide sufficient contrast range.

For perceptually uniform color ramps, use `oklch` or `hsl` instead of hex. The `oklch` color space produces smoother gradients and more consistent perceived brightness across a scale:

```css
@theme {
  --color-brand-50: oklch(0.97 0.02 250);
  --color-brand-500: oklch(0.62 0.18 250);
  --color-brand-900: oklch(0.30 0.10 250);
}
```

For dark mode, define both light and dark tokens. Use CSS custom properties toggled by a `.dark` class or `prefers-color-scheme` media query to swap between them. This keeps the utility classes identical across themes (`bg-surface`, `text-content`) while the underlying values change.

Opacity modifier support works automatically with extended colors. Writing `bg-brand-500/75` applies the brand color at 75% opacity without any additional configuration.

## Custom Spacing and Sizing

Extend the spacing scale to cover project-specific layout needs. Add values at the gaps Tailwind does not cover by default, such as `18` (4.5rem) or `88` (22rem). Every value added to the spacing scale becomes available across all spacing-related utilities: margin, padding, gap, space, inset, and translate.

Negative value support is automatic. After adding `'18': '4.5rem'` to the spacing scale, `-m-18` works immediately with no extra setup.

For width and max-width tokens in v4, use the `--width-*` and `--max-width-*` namespaces to register sizing values that appear in `w-*` and `max-w-*` utilities respectively.

## Custom Typography

Register font families with complete fallback stacks to prevent layout shift when custom fonts load. Always include a generic family (`sans-serif`, `serif`, `monospace`) as the final fallback:

```javascript
fontFamily: {
  heading: ['Inter', 'system-ui', 'sans-serif'],
  body: ['Source Sans 3', 'Georgia', 'serif'],
}
```

Define custom font sizes with coupled line-height values to maintain vertical rhythm. In v3, pass an array with the size and an options object:

```javascript
fontSize: {
  'display': ['4rem', { lineHeight: '1.1', letterSpacing: '-0.02em' }],
  'caption': ['0.75rem', { lineHeight: '1.4' }],
}
```

In v4, express the same pairing with companion custom properties:

```css
@theme {
  --font-size-display: 4rem;
  --font-size-display--line-height: 1.1;
  --font-size-display--letter-spacing: -0.02em;
  --font-size-caption: 0.75rem;
  --font-size-caption--line-height: 1.4;
}
```

## Design Token Integration

Map design system tokens from Figma or other tools directly into the Tailwind configuration to maintain a single source of truth. Adopt a naming convention that reflects purpose:

- `brand-*` for primary brand colors
- `surface-*` for background colors (surface-primary, surface-secondary)
- `content-*` for text and icon colors (content-primary, content-muted)
- `border-*` for border and divider colors

Generate the Tailwind config programmatically from a token JSON file exported by the design tool. Parse the JSON at build time and spread the values into the appropriate theme keys:

```javascript
import tokens from './design-tokens.json';

export default {
  theme: {
    extend: {
      colors: tokens.colors,
      fontFamily: tokens.fonts,
      spacing: tokens.spacing,
    },
  },
} satisfies Config;
```

Keep tokens in sync by integrating the export step into CI. When a designer publishes updated tokens, the pipeline regenerates the JSON file, and any drift between design and code surfaces as a diff in the pull request.

## Presets

Create a shared preset file to enforce consistency across multiple projects or packages within a monorepo. A preset is a standard Tailwind configuration object exported from its own module:

```javascript
// tailwind-preset.js
export default {
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#f0f5ff',
          500: '#3b82f6',
          900: '#1e3a5f',
        },
      },
      fontFamily: {
        heading: ['Inter', 'sans-serif'],
        body: ['Source Sans 3', 'sans-serif'],
      },
    },
  },
  plugins: [/* shared plugins */],
};
```

Consume the preset in any project's configuration:

```javascript
presets: [require('./tailwind-preset')]
```

When a project specifies `theme.extend` alongside a preset, the extensions merge with the preset's theme. Direct `theme` overrides in the consuming project replace the preset's values for that key.

In v4, achieve the same shared configuration by extracting the `@theme` block into a separate CSS file and importing it in each project's main stylesheet with `@import "./shared-theme.css"`.

## Plugin Basics

Tailwind plugins extend the framework with custom utilities, components, base styles, and variants. The plugin API provides several registration functions:

- `addUtilities` -- register custom utility classes that work with responsive and state variants
- `addComponents` -- register component-level classes (cards, buttons, badges)
- `addVariant` -- register custom variants like `hocus` (hover + focus)
- `addBase` -- inject base/reset styles into the `@layer base`

Official first-party plugins cover common needs:

- `@tailwindcss/typography` -- beautiful prose defaults for rendered markdown and CMS content
- `@tailwindcss/forms` -- opinionated form element resets with better default styling
- `@tailwindcss/container-queries` -- container query variants (`@container`, `@lg:`)
- `@tailwindcss/aspect-ratio` -- aspect ratio utilities for browsers lacking native support

For a deep dive into writing custom plugins, see [plugin-development.md](references/plugin-development.md).

## Content Paths

Configure content paths to tell Tailwind which files to scan for class names. Use glob patterns that cover every file containing Tailwind classes:

```javascript
content: ['./src/**/*.{js,ts,jsx,tsx,mdx}']
```

Include paths to component library packages, CMS template files, and MDX content directories. Tailwind automatically excludes `node_modules`, so there is no need to add a negation pattern for it. Optionally exclude test files to reduce scan time if tests never contain production class names.

For classes that are assembled dynamically at runtime and cannot appear as complete strings in source files, use the `safelist` option:

```javascript
safelist: ['bg-red-500', 'bg-green-500', 'bg-blue-500']
```

Limit safelist usage to genuinely dynamic cases. Overuse defeats the purpose of Tailwind's file-size optimization through tree-shaking.

## IDE and Tooling

Install the **Tailwind CSS IntelliSense** extension for VS Code to gain autocomplete for every utility class, color previews in the editor gutter, hover documentation showing the generated CSS, and linting for common mistakes like conflicting classes.

Add `prettier-plugin-tailwindcss` to enforce a consistent class ordering across the team. The plugin sorts classes following Tailwind's recommended order (layout, box model, typography, visual, interactive) automatically on save:

```bash
npm install -D prettier-plugin-tailwindcss
```

For ESLint integration, use `eslint-plugin-tailwindcss` to validate class names against the project's configuration, catch typos, and flag deprecated utilities.

## References

- [Plugin Development](references/plugin-development.md) -- detailed guide to writing custom Tailwind plugins
