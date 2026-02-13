# Tailwind Plugin Development

This reference covers the full plugin API, patterns for creating custom utilities, components, and variants, the third-party ecosystem, and migration considerations for v4.

## Plugin API

Build every Tailwind plugin using the `plugin` function from `tailwindcss/plugin`. This function accepts a callback that receives a destructured helper object:

```javascript
const plugin = require('tailwindcss/plugin');

module.exports = plugin(function({
  addUtilities,
  addComponents,
  addBase,
  addVariant,
  matchUtilities,
  theme,
  e,
  config,
}) {
  // plugin logic here
});
```

Each helper serves a distinct purpose:

- **`addUtilities(styles)`** -- Register utility classes that live in the `@layer utilities` layer. These classes are purge-safe and support responsive, hover, focus, and other state variants by default.
- **`addComponents(styles)`** -- Register component classes in the `@layer components` layer. Use this for multi-property patterns like cards, buttons, or badges that represent a single UI element.
- **`addBase(styles)`** -- Inject styles into the `@layer base` layer. Use this for global resets, custom font-face declarations, or default element styling.
- **`addVariant(name, definition)`** -- Register a custom variant selector or at-rule that can prefix any utility class.
- **`matchUtilities(utilities, options)`** -- Register dynamic utilities that accept arbitrary values from the theme or via square bracket notation.
- **`theme(path)`** -- Read values from the resolved Tailwind theme. Call `theme('colors.brand.500')` to retrieve a specific token.
- **`e(className)`** -- Escape a string for safe use as a CSS class name. Essential when generating classes from user-provided or dynamic values that may contain special characters.
- **`config(path)`** -- Read values from the raw Tailwind configuration object.

The `plugin` function also accepts a second argument for default theme extensions that the plugin provides:

```javascript
module.exports = plugin(
  function({ addUtilities, theme }) {
    // use theme('customKey') here
  },
  {
    theme: {
      extend: {
        customKey: {
          sm: '0.5rem',
          md: '1rem',
          lg: '2rem',
        },
      },
    },
  }
);
```

This pattern lets plugin consumers override the defaults through their own `theme.extend` without modifying the plugin source.

## Creating Custom Utilities

Register static utility classes with `addUtilities`. Pass an object where each key is the class selector (including the leading dot) and the value is a standard CSS-in-JS style object:

```javascript
const plugin = require('tailwindcss/plugin');

module.exports = plugin(function({ addUtilities }) {
  addUtilities({
    '.text-balance': {
      'text-wrap': 'balance',
    },
    '.scrollbar-hide': {
      '-ms-overflow-style': 'none',
      'scrollbar-width': 'none',
      '&::-webkit-scrollbar': {
        display: 'none',
      },
    },
    '.content-auto': {
      'content-visibility': 'auto',
    },
    '.drag-none': {
      '-webkit-user-drag': 'none',
      'user-drag': 'none',
    },
  });
});
```

Nest pseudo-elements and pseudo-classes using the `&` selector, just like in Sass or PostCSS nesting. Tailwind resolves these at build time.

Group related utilities into a single plugin for easier distribution. For example, bundle all scrollbar-related utilities (`.scrollbar-hide`, `.scrollbar-thin`, `.scrollbar-auto`) into one plugin rather than scattering them across separate files.

To restrict variant generation, pass an options object as the second argument to `addUtilities`:

```javascript
addUtilities(styles, {
  respectPrefix: true,
  respectImportant: true,
});
```

Both options default to `true`. Set `respectPrefix` to `false` only when the utility must work without the user's configured prefix (rare).

## Dynamic Utilities with matchUtilities

Use `matchUtilities` to generate utilities that accept values from a theme scale or arbitrary values via square bracket notation. This is the mechanism behind utilities like `p-4` or `text-[1.25rem]`.

Define the utility name as a key in the first argument. The value is a function that receives the resolved value and returns a CSS-in-JS object:

```javascript
const plugin = require('tailwindcss/plugin');

module.exports = plugin(
  function({ matchUtilities, theme }) {
    matchUtilities(
      {
        'fluid-gap': (value) => ({
          gap: value,
        }),
      },
      {
        values: theme('fluidSpacing'),
        type: ['length', 'percentage'],
      }
    );
  },
  {
    theme: {
      extend: {
        fluidSpacing: {
          sm: 'clamp(0.5rem, 1vw, 1rem)',
          md: 'clamp(1rem, 2vw, 2rem)',
          lg: 'clamp(1.5rem, 3vw, 3rem)',
          xl: 'clamp(2rem, 4vw, 4rem)',
        },
      },
    },
  }
);
```

After registration, use the utility as `fluid-gap-sm`, `fluid-gap-lg`, or with an arbitrary value like `fluid-gap-[clamp(1rem,2vw,3rem)]`.

The `type` option in the second argument constrains what kinds of arbitrary values are accepted. Common types include `length`, `percentage`, `color`, `number`, `angle`, and `url`. Specify multiple types to allow flexibility.

For negative value support, add `supportsNegativeValues: true` to the options:

```javascript
matchUtilities(
  {
    'skew-x': (value) => ({
      transform: `skewX(${value})`,
    }),
  },
  {
    values: theme('skew'),
    type: ['angle'],
    supportsNegativeValues: true,
  }
);
```

This enables both `skew-x-6` and `-skew-x-6` without any extra logic.

## Creating Custom Variants

Register custom variants with `addVariant` to extend the selector and at-rule system. Each variant receives a name and a selector template or array of templates:

```javascript
const plugin = require('tailwindcss/plugin');

module.exports = plugin(function({ addVariant }) {
  // Combine hover and focus into a single variant
  addVariant('hocus', ['&:hover', '&:focus']);

  // Feature detection via @supports
  addVariant('supports-grid', '@supports (display: grid)');
  addVariant('supports-backdrop', '@supports (backdrop-filter: blur(1px))');

  // Target direct children
  addVariant('child', '& > *');
  addVariant('child-hover', '& > *:hover');

  // Target a specific data attribute
  addVariant('data-active', '&[data-active="true"]');

  // Group and peer patterns
  addVariant('group-open', ':merge(.group)[open] &');

  // Print media query
  addVariant('print', '@media print');

  // Prefers reduced motion shorthand
  addVariant('motion-reduce', '@media (prefers-reduced-motion: reduce)');
});
```

Use the `&` character as a placeholder for the selector that the variant wraps. When an array is passed, Tailwind generates a rule for each selector, making the utility apply when any of the conditions match.

After registering a variant, prefix any utility class with it: `hocus:text-blue-500`, `supports-grid:grid`, `child:mb-4`. Variants compose with each other and with built-in variants, so `md:hocus:text-blue-500` works as expected.

For at-rule variants (like `@supports` or `@media`), pass the full at-rule string. Tailwind wraps the generated utility inside that at-rule automatically.

## Creating Component Classes

Use `addComponents` for multi-property class definitions that represent discrete UI elements. Unlike utilities (which apply a single CSS property), components bundle several properties into a meaningful abstraction:

```javascript
const plugin = require('tailwindcss/plugin');

module.exports = plugin(function({ addComponents, theme }) {
  addComponents({
    '.btn': {
      display: 'inline-flex',
      alignItems: 'center',
      justifyContent: 'center',
      padding: `${theme('spacing.2')} ${theme('spacing.4')}`,
      fontSize: theme('fontSize.sm[0]'),
      fontWeight: theme('fontWeight.semibold'),
      borderRadius: theme('borderRadius.lg'),
      transitionProperty: 'background-color, border-color, color',
      transitionDuration: '150ms',
    },
    '.btn-primary': {
      backgroundColor: theme('colors.blue.600'),
      color: theme('colors.white'),
      '&:hover': {
        backgroundColor: theme('colors.blue.700'),
      },
    },
    '.card': {
      backgroundColor: theme('colors.white'),
      borderRadius: theme('borderRadius.xl'),
      padding: theme('spacing.6'),
      boxShadow: theme('boxShadow.md'),
    },
  });
});
```

Choose `addComponents` over `addUtilities` when the class:

- Combines three or more CSS properties into a single concept
- Represents a recognizable UI pattern (button, card, badge, alert)
- Benefits from a stable, semantic name rather than a composition of utilities

Choose `addUtilities` when the class:

- Maps to one or two CSS properties
- Is designed to be composed with other utilities
- Has no inherent semantic meaning beyond its CSS effect

Component classes registered with `addComponents` support responsive and state variants the same way utilities do. Apply `md:card` or `hover:btn-primary` without any additional configuration.

## Third-Party Plugin Ecosystem

### Official Plugins

**@tailwindcss/typography** -- Apply the `prose` class to any container holding rendered HTML or markdown content. It provides carefully tuned typographic defaults for headings, paragraphs, lists, code blocks, tables, and more. Customize the styles through the `typography` theme key or by overriding specific elements with modifier classes like `prose-lg` or `prose-invert`.

**@tailwindcss/forms** -- Reset form elements (inputs, selects, textareas, checkboxes, radios) to a consistent baseline that is easy to style with utilities. Install it and all form elements receive sensible defaults without any class additions. Use the `strategy: 'class'` option to apply resets only when a `.form-input`, `.form-select`, or similar class is present.

**@tailwindcss/container-queries** -- Enable container query support with `@container` classes. Mark an element as a container with the `@container` class, then style descendants with size-based variants like `@lg:grid-cols-2` or `@md:flex-row`. Name containers with `@container/sidebar` and reference them with `@lg/sidebar:hidden`.

**tailwindcss-animate** -- Provide a comprehensive set of animation utilities: fade, slide, zoom, spin, and more. Each animation supports duration, delay, fill-mode, and direction modifiers. Pair with the `animate-in` and `animate-out` utilities for enter/leave transitions.

### Community Plugins

**tailwind-merge** -- Not a Tailwind plugin but a runtime utility for intelligently merging Tailwind class strings. It resolves conflicts where later classes should override earlier ones (e.g., `twMerge('px-4 px-6')` returns `'px-6'`). Essential for component libraries that accept className props.

When evaluating community plugins, apply these criteria:

- Check maintenance status: look for recent commits and active issue triage
- Verify compatibility with the project's Tailwind version (v3 vs v4)
- Review bundle size impact, especially for client-side plugins
- Confirm the plugin respects the `prefix` and `important` configuration options
- Prefer plugins with TypeScript type definitions

## v4 Plugin Changes

Tailwind CSS v4 introduces a CSS-first approach to plugins alongside the existing JavaScript API.

### The @plugin Directive

Load JavaScript plugins directly from CSS using the `@plugin` directive instead of the `plugins` array in a config file:

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
@plugin "./custom-plugin.js";
```

This keeps all Tailwind configuration -- theme, plugins, and imports -- in a single CSS file, eliminating the need for `tailwind.config.js` in many projects.

### CSS-Based Extensions

For simple customizations that would previously require a plugin, use native CSS features within the Tailwind layers:

```css
@import "tailwindcss";

@utility text-balance {
  text-wrap: balance;
}

@utility scrollbar-hide {
  -ms-overflow-style: none;
  scrollbar-width: none;
  &::-webkit-scrollbar {
    display: none;
  }
}

@variant hocus (&:hover, &:focus);
```

The `@utility` directive registers a custom utility class that participates in Tailwind's variant system automatically. The `@variant` directive registers a custom variant. Both approaches require zero JavaScript.

### Migrating v3 Plugins to v4

Follow these steps to migrate an existing v3 plugin:

1. Determine whether the plugin's logic can be expressed purely in CSS using `@utility` and `@variant`. If so, replace the JavaScript plugin entirely with CSS directives.
2. For plugins that require dynamic logic (reading theme values, generating utilities from configuration), keep the JavaScript implementation and load it with `@plugin "./path-to-plugin.js"`.
3. Update any `theme()` function calls to use CSS custom properties instead. Replace `theme('colors.blue.500')` with `var(--color-blue-500)`.
4. Replace `addUtilities` calls with `@utility` directives where the utility is static (no dynamic values).
5. Keep `matchUtilities` for utilities that need to accept arbitrary values or read from a theme scale.
6. Test all registered variants, as the selector format may differ slightly in v4's engine.
7. Remove the plugin from `tailwind.config.js` `plugins` array and add the corresponding `@plugin` directive in CSS.

### Plugin Configuration in v4

Pass options to plugins loaded via `@plugin` using a CSS-compatible syntax:

```css
@plugin "@tailwindcss/typography" {
  className: prose;
}
```

For custom plugins that accept options, export a function that receives the options object:

```javascript
const plugin = require('tailwindcss/plugin');

module.exports = plugin.withOptions(
  function(options = {}) {
    return function({ addComponents, theme }) {
      const prefix = options.prefix || 'ui';
      addComponents({
        [`.${prefix}-button`]: {
          padding: `${theme('spacing.2')} ${theme('spacing.4')}`,
          borderRadius: theme('borderRadius.md'),
        },
      });
    };
  },
  function(options = {}) {
    return {
      theme: {
        extend: {
          // default theme extensions
        },
      },
    };
  }
);
```

The `plugin.withOptions` pattern separates plugin logic (first function) from default configuration (second function). This allows consumers to customize the plugin's behavior without forking it.

### Testing Plugins

Validate plugin output by running Tailwind's build against a test stylesheet that uses the registered classes. Create a minimal CSS input file:

```css
@import "tailwindcss";
@plugin "./my-plugin.js";

.test {
  @apply text-balance scrollbar-hide;
}
```

Run the Tailwind CLI and inspect the output CSS to confirm the expected declarations appear. For automated testing, use a test runner to compare the generated CSS against a snapshot:

```javascript
const postcss = require('postcss');
const tailwindcss = require('tailwindcss');

async function generateCSS(input) {
  const result = await postcss([tailwindcss]).process(input, {
    from: undefined,
  });
  return result.css;
}

test('text-balance utility generates correct CSS', async () => {
  const css = await generateCSS(`
    @tailwind utilities;
    .test { @apply text-balance; }
  `);
  expect(css).toContain('text-wrap: balance');
});
```

Adopt snapshot testing for complex plugins that generate many utilities. Update snapshots deliberately when the plugin's output changes intentionally, and treat unexpected snapshot changes as regressions.
