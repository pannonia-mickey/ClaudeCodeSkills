---
name: Tailwind Mastery
description: This skill should be used when the user asks about "Tailwind utility classes", "Tailwind responsive design", "Tailwind dark mode", "Tailwind spacing", "Tailwind typography", "Tailwind colors", "hover focus states", "Tailwind arbitrary values", "Tailwind @apply", or "utility-first CSS". It covers core Tailwind CSS v3.4+ and v4 fundamentals including the utility-first methodology, spacing scale, typography, color system, layout with Flexbox and Grid, responsive breakpoints, dark mode, state variants, and arbitrary values.
---

# Tailwind Mastery

## Utility-First Philosophy

To understand Tailwind CSS, start with its core principle: apply small, single-purpose utility classes directly in HTML or JSX markup rather than writing custom CSS in separate stylesheets. Each utility does exactly one thing. `flex` sets `display: flex`. `pt-4` sets `padding-top: 1rem`. `text-center` sets `text-align: center`. To build a component, compose these utilities together on the element itself.

To contrast with traditional CSS, consider a notification card. In a traditional approach, define a `.notification-card` class in a stylesheet and write out every property: display, padding, border-radius, background, font-size, and color. With the utility-first approach, apply `flex items-center gap-4 rounded-lg bg-white p-4 text-sm text-gray-700 shadow-md` directly on the element. The result is the same visual output, but the utility-first approach eliminates the need to name things, switch between files, or manage specificity conflicts.

To decide when to extract utilities into a reusable class, follow this rule: extract only when the exact same combination of utilities repeats across many files AND the pattern cannot be captured as a framework component (React, Vue, Svelte, etc.). In component-based architectures, prefer creating a component that encapsulates the markup and its utilities. Reserve `@apply` extraction for truly global patterns like base heading styles or shared link styles that span the entire application.

## Spacing and Sizing

To apply spacing in Tailwind, use its predefined scale that maps numeric values to `rem` units. The default scale includes: `0` (0), `px` (1px), `0.5` (0.125rem), `1` (0.25rem), `1.5` (0.375rem), `2` (0.5rem), `2.5` (0.625rem), `3` (0.75rem), `3.5` (0.875rem), `4` (1rem), `5` (1.25rem), `6` (1.5rem), `7` (1.75rem), `8` (2rem), `9` (2.25rem), `10` (2.5rem), `11` (2.75rem), `12` (3rem), `14` (3.5rem), `16` (4rem), `20` (5rem), `24` (6rem), `28` (7rem), `32` (8rem), `36` (9rem), `40` (10rem), `44` (11rem), `48` (12rem), `52` (13rem), `56` (14rem), `60` (15rem), `64` (16rem), `72` (18rem), `80` (20rem), `96` (24rem).

To set padding, use `p-{value}` for all sides, `px-{value}` for horizontal, `py-{value}` for vertical, or `pt-`, `pr-`, `pb-`, `pl-` for individual sides. To set margin, use the same pattern with `m-` instead of `p-`. To center a block element horizontally, use `mx-auto`. To apply negative margins, prefix with a dash: `-mt-2` applies `margin-top: -0.5rem`.

To control width and height, use `w-` and `h-` prefixes. Common values include `w-full` (100%), `w-1/2` (50%), `w-screen` (100vw), `h-screen` (100vh), `min-h-0`, and `max-w-lg` (32rem). To set both width and height to the same value simultaneously, use the `size-` utility: `size-10` sets both width and height to 2.5rem.

**Quick Reference Table -- Common Spacing Values:**

| Class   | Value     | Pixels |
|---------|-----------|--------|
| `0`     | 0         | 0px    |
| `1`     | 0.25rem   | 4px    |
| `2`     | 0.5rem    | 8px    |
| `3`     | 0.75rem   | 12px   |
| `4`     | 1rem      | 16px   |
| `6`     | 1.5rem    | 24px   |
| `8`     | 2rem      | 32px   |
| `12`    | 3rem      | 48px   |
| `16`    | 4rem      | 64px   |
| `24`    | 6rem      | 96px   |
| `32`    | 8rem      | 128px  |
| `48`    | 12rem     | 192px  |
| `64`    | 16rem     | 256px  |
| `96`    | 24rem     | 384px  |

## Typography

To set the font family, use `font-sans` (the default system font stack), `font-serif`, or `font-mono`. To customize these families, extend the theme configuration with project-specific typefaces.

To control font size, use the `text-` prefix. Each size maps to a `font-size` and a default `line-height`, so there is no need to set line-height separately in most cases. The standard scale includes `text-xs` (0.75rem/1rem), `text-sm` (0.875rem/1.25rem), `text-base` (1rem/1.5rem), `text-lg` (1.125rem/1.75rem), `text-xl` (1.25rem/1.75rem), `text-2xl` (1.5rem/2rem), `text-3xl` (1.875rem/2.25rem), `text-4xl` (2.25rem/2.5rem), `text-5xl` (3rem/1), `text-6xl` (3.75rem/1), `text-7xl` (4.5rem/1), `text-8xl` (6rem/1), and `text-9xl` (8rem/1).

To set font weight, use `font-thin` (100), `font-extralight` (200), `font-light` (300), `font-normal` (400), `font-medium` (500), `font-semibold` (600), `font-bold` (700), `font-extrabold` (800), or `font-black` (900).

To align text, use `text-left`, `text-center`, `text-right`, or `text-justify`. To apply text color, use `text-{color}-{shade}` such as `text-gray-900` or `text-blue-600`. To add decoration, use `underline`, `overline`, `line-through`, or `no-underline`. To transform text case, use `uppercase`, `lowercase`, `capitalize`, or `normal-case`. To handle text overflow, use `truncate` (which combines overflow-hidden, text-overflow-ellipsis, and white-space-nowrap), `text-ellipsis`, or `text-clip`.

To style rich text content from a CMS or markdown, use the official Typography plugin with the `prose` class. Apply `prose prose-lg` on the container wrapping the HTML content. To support dark mode, add `dark:prose-invert`. The plugin provides sensible default styles for headings, paragraphs, lists, blockquotes, tables, code blocks, and more without requiring individual utility classes on each element.

## Color System

To apply colors in Tailwind, use the pattern `{property}-{color}-{shade}`. The property can be `text`, `bg`, `border`, `ring`, `fill`, `stroke`, `decoration`, `outline`, `accent`, `caret`, `shadow`, or `divide`. The default palette includes `slate`, `gray`, `zinc`, `neutral`, `stone`, `red`, `orange`, `amber`, `yellow`, `lime`, `green`, `emerald`, `teal`, `cyan`, `sky`, `blue`, `indigo`, `violet`, `purple`, `fuchsia`, `pink`, and `rose`. Each color has shades from `50` (lightest) through `950` (darkest).

To apply color with opacity, use the slash modifier: `bg-red-500/50` renders the red-500 background at 50% opacity. Any integer from 0 to 100 works, as do arbitrary values like `bg-red-500/[.33]`. This replaces the older `bg-opacity-` utilities.

To reference the current text color for SVG fills or strokes, use `text-current`, `fill-current`, or `stroke-current`.

To use a one-off color not in the palette, use arbitrary value syntax: `text-[#1a2b3c]` or `bg-[oklch(0.5_0.2_240)]`. For repeated use of a custom color, extend the theme configuration instead.

## Layout with Flexbox and Grid

To create a flex layout, apply `flex` on the container. To set direction, use `flex-row` (default) or `flex-col`. To align items on the cross axis, use `items-start`, `items-center`, `items-end`, `items-baseline`, or `items-stretch`. To distribute items on the main axis, use `justify-start`, `justify-center`, `justify-end`, `justify-between`, or `justify-around`. To add spacing between children, use `gap-4` (or `gap-x-` / `gap-y-` for axis-specific gaps). To allow a child to grow and fill available space, apply `flex-1`. To prevent a child from shrinking, apply `flex-shrink-0`. To enable wrapping, add `flex-wrap`.

To create a grid layout, apply `grid` on the container. To define columns, use `grid-cols-3` for three equal columns or `grid-cols-[1fr_2fr]` for custom track sizing with arbitrary values. To span a child across columns, use `col-span-2`. To set gap between grid items, use `gap-6`.

To build a responsive card grid, combine grid with breakpoint prefixes:

```html
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
  <div class="rounded-lg bg-white p-6 shadow">Card 1</div>
  <div class="rounded-lg bg-white p-6 shadow">Card 2</div>
  <div class="rounded-lg bg-white p-6 shadow">Card 3</div>
</div>
```

This creates a single column on mobile, two columns at the `sm` breakpoint, and three columns at `lg`.

## Responsive Design

To build responsive interfaces, use Tailwind's mobile-first breakpoint system. The default breakpoints are `sm:` (640px), `md:` (768px), `lg:` (1024px), `xl:` (1280px), and `2xl:` (1536px). Each prefix applies its styles at that minimum width and above.

To apply styles, write the base (mobile) styles unprefixed and layer on larger-screen overrides with breakpoint prefixes. For example, `text-sm md:text-base lg:text-lg` renders small text on mobile, base size from 768px, and large text from 1024px.

To use container queries instead of viewport-based breakpoints, apply `@container` on a parent element and use `@sm:`, `@md:`, `@lg:` etc. on children. Container queries respond to the parent element's width rather than the viewport, making components truly portable across different layout contexts.

For a comprehensive guide to breakpoint customization, fluid typography, and responsive image techniques, see [responsive-design.md](references/responsive-design.md).

## Dark Mode

To support dark mode, use the `dark:` variant prefix. For example, `bg-white dark:bg-gray-900` sets a white background in light mode and a dark gray background in dark mode. Apply `dark:` to any utility: `text-gray-900 dark:text-gray-100`, `border-gray-200 dark:border-gray-700`, `shadow-lg dark:shadow-gray-900/30`.

To configure dark mode detection in Tailwind v3, choose between two strategies. The `class` strategy activates dark mode when the `dark` class is present on the `<html>` element, giving full programmatic control. The `media` strategy uses the operating system's `prefers-color-scheme` preference. Set this in `tailwind.config.js` with `darkMode: 'class'` or `darkMode: 'media'`.

To configure dark mode in Tailwind v4, note that it uses `prefers-color-scheme` by default. To switch to a class-based approach, use `@custom-variant dark (&:where(.dark, .dark *))` in the CSS file.

To stack dark mode with other variants, chain them together: `dark:hover:bg-gray-700` applies on hover only in dark mode. The order of variants does not change specificity; Tailwind handles the cascade internally.

## State Variants

To style interactive states, apply variant prefixes. Use `hover:bg-blue-600` for mouse hover, `focus:ring-2` for focus, `active:scale-95` for active/pressed state, and `disabled:opacity-50` for disabled elements. To style positional pseudo-classes, use `first:pt-0`, `last:pb-0`, `odd:bg-gray-50`, and `even:bg-white`.

To provide keyboard-accessible focus indicators without showing them on mouse click, use `focus-visible:ring-2 focus-visible:ring-blue-500`. This applies focus styles only when the browser determines the focus should be visible (typically keyboard navigation).

To style a child based on a parent's state, add the `group` class to the parent and use `group-hover:`, `group-focus:`, or `group-active:` on the child. To style a sibling based on another sibling's state, apply the `peer` class to the trigger element and use `peer-checked:`, `peer-invalid:`, `peer-focus:`, or `peer-placeholder-shown:` on the target.

To style based on data attributes, use the `data-` variant: `data-[state=open]:rotate-180` applies rotation when `data-state="open"` is present. This integrates well with headless UI libraries that communicate state via data attributes.

To combine multiple variants, stack them left to right: `dark:md:hover:bg-fuchsia-600` applies on hover, at medium breakpoint and above, in dark mode. Tailwind generates the correct compound selector regardless of stacking order.

## Arbitrary Values and Properties

To use a one-off value not in the default scale, wrap it in square brackets. Apply arbitrary values to any utility: `w-[calc(100%-2rem)]` for a calculated width, `top-[117px]` for a specific offset, `grid-cols-[1fr_auto_1fr]` for custom grid tracks, `text-[22px]` for a non-standard font size, `bg-[#243c5a]` for a hex color.

To apply a CSS property that has no corresponding Tailwind utility, use arbitrary property syntax: `[mask-type:luminance]` generates `mask-type: luminance`. This covers edge cases without leaving the utility-first workflow.

To target specific selectors or contexts, use arbitrary variants: `[&>svg]:w-4` targets direct SVG children, `[&_p]:mt-4` targets descendant paragraphs, `[@supports(backdrop-filter:blur(0))]:backdrop-blur` applies only when the browser supports backdrop-filter.

To maintain consistency, use arbitrary values sparingly. When the same arbitrary value appears in multiple places, extend the theme configuration instead. Repeated arbitrary values signal that the design system should incorporate that value as a first-class token.

## The @apply Directive

To extract a set of utilities into a traditional CSS class, use the `@apply` directive:

```css
.btn-primary {
  @apply inline-flex items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-sm font-semibold text-white shadow-sm hover:bg-blue-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600;
}
```

To decide when `@apply` is appropriate, consider these cases. Use it for utility patterns shared across many files where a component abstraction is not available, such as global base heading styles, shared link styles in a multi-page server-rendered application, or third-party library overrides. Use it when the same utility combination must appear in plain CSS files (e.g., global stylesheet resets).

To avoid misuse, do not reach for `@apply` in component-based frameworks like React, Vue, or Svelte where the component itself serves as the abstraction. Do not use it for one-off styles that appear only once. Do not use it when it obscures the design intent by hiding utilities behind opaque class names. The primary value of utility-first CSS is co-locating style with structure; `@apply` should not undermine that benefit.

To note the status in Tailwind v4: `@apply` remains supported and functional. However, the Tailwind team continues to recommend component-level extraction as the preferred abstraction method. Reach for `@apply` as a pragmatic tool when component abstraction is not feasible.

## References

- For Flexbox vs Grid deep-dive, common layout recipes, container query patterns, and positioning techniques, see [layout-patterns.md](references/layout-patterns.md).
- For breakpoint customization, fluid typography, responsive image techniques, container queries in depth, and print styles, see [responsive-design.md](references/responsive-design.md).
