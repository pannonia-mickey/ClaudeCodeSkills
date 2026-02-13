---
name: Tailwind Component Patterns
description: This skill should be used when the user asks about "Tailwind button", "Tailwind card", "Tailwind navbar", "Tailwind form", "Tailwind modal", "Tailwind table styling", "Tailwind alert", "UI component Tailwind", "component extraction", or "Tailwind @apply patterns". It covers building common UI components with Tailwind utilities including buttons, cards, navigation, forms, modals, tables, alerts, and the decision between @apply extraction and framework-level components.
---

# Tailwind Component Patterns

## Button Variants

To build a primary button, apply a solid background with hover darkening, white text, and a visible focus ring:

```html
<button class="bg-blue-600 hover:bg-blue-700 text-white font-medium py-2 px-4 rounded-lg transition-colors focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600">
  Primary Button
</button>
```

To create a secondary outline button, replace the solid background with a transparent one and add a border:

```html
<button class="bg-transparent hover:bg-gray-50 text-gray-700 font-medium py-2 px-4 rounded-lg border border-gray-300 transition-colors focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-gray-400">
  Secondary Button
</button>
```

To indicate a destructive action, use red tones for the danger variant:

```html
<button class="bg-red-600 hover:bg-red-700 text-white font-medium py-2 px-4 rounded-lg transition-colors focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-red-600">
  Delete
</button>
```

To handle the disabled state, append `disabled:opacity-50 disabled:cursor-not-allowed` to any button variant. These utilities gray out the button and replace the pointer with a not-allowed cursor when the `disabled` attribute is present.

To show a loading state, insert a spinner SVG before the label and apply `animate-spin` to the icon:

```html
<button class="bg-blue-600 text-white font-medium py-2 px-4 rounded-lg inline-flex items-center gap-2" disabled>
  <svg class="animate-spin h-4 w-4 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
  </svg>
  Processing...
</button>
```

To build an icon-only button, make it square by applying equal width and height and centering the icon:

```html
<button class="h-10 w-10 inline-flex items-center justify-center rounded-lg bg-gray-100 hover:bg-gray-200 text-gray-700 transition-colors">
  <!-- icon SVG -->
</button>
```

To create a button group, wrap buttons in a `flex` parent and remove interior rounding. Apply `first:rounded-l-lg last:rounded-r-lg rounded-none` to each child so only the outermost corners are rounded:

```html
<div class="inline-flex">
  <button class="px-4 py-2 bg-white border border-gray-300 text-sm font-medium text-gray-700 hover:bg-gray-50 first:rounded-l-lg last:rounded-r-lg rounded-none">Left</button>
  <button class="px-4 py-2 bg-white border-t border-b border-gray-300 text-sm font-medium text-gray-700 hover:bg-gray-50 rounded-none">Center</button>
  <button class="px-4 py-2 bg-white border border-gray-300 text-sm font-medium text-gray-700 hover:bg-gray-50 first:rounded-l-lg last:rounded-r-lg rounded-none">Right</button>
</div>
```

## Card Patterns

To build a basic card, combine rounded corners, a subtle border, white background, soft shadow, and padding:

```html
<div class="rounded-lg border border-gray-200 bg-white shadow-sm p-6">
  <h3 class="text-lg font-semibold text-gray-900">Card Title</h3>
  <p class="mt-2 text-gray-600">Card content goes here.</p>
</div>
```

To create a card with distinct header, body, and footer sections, use `divide-y` on the parent to insert horizontal rules between children:

```html
<div class="rounded-lg border border-gray-200 bg-white shadow-sm divide-y divide-gray-200">
  <div class="px-6 py-4">
    <h3 class="text-lg font-semibold text-gray-900">Header</h3>
  </div>
  <div class="px-6 py-4">
    <p class="text-gray-600">Body content with details.</p>
  </div>
  <div class="px-6 py-4 bg-gray-50 rounded-b-lg">
    <button class="text-sm text-blue-600 hover:text-blue-700 font-medium">Action</button>
  </div>
</div>
```

To lay out a horizontal card with an image on the left and content on the right, use `flex` on the parent:

```html
<div class="flex rounded-lg border border-gray-200 bg-white shadow-sm overflow-hidden">
  <img src="/image.jpg" alt="" class="w-48 object-cover" />
  <div class="p-6">
    <h3 class="text-lg font-semibold text-gray-900">Horizontal Card</h3>
    <p class="mt-2 text-gray-600">Content beside the image.</p>
  </div>
</div>
```

To make a card interactive, add hover shadow elevation and a pointer cursor:

```html
<div class="rounded-lg border border-gray-200 bg-white shadow-sm p-6 hover:shadow-lg transition-shadow cursor-pointer">
  <!-- card content -->
</div>
```

To support dark mode, append dark variants to the card container: `dark:bg-gray-800 dark:border-gray-700`. Apply corresponding dark text colors to children such as `dark:text-gray-100` for headings and `dark:text-gray-300` for body text.

## Navigation

To build a horizontal navbar, use `flex` with vertical centering and horizontal space-between:

```html
<nav class="flex items-center justify-between px-6 h-16 bg-white border-b border-gray-200">
  <a href="/" class="text-xl font-bold text-gray-900">Logo</a>
  <div class="hidden lg:flex items-center gap-6">
    <a href="/about" class="text-sm font-medium text-gray-600 hover:text-gray-900">About</a>
    <a href="/pricing" class="text-sm font-medium text-gray-600 hover:text-gray-900">Pricing</a>
    <a href="/contact" class="text-sm font-medium text-gray-600 hover:text-gray-900">Contact</a>
  </div>
  <button class="lg:hidden p-2 text-gray-600 hover:text-gray-900" aria-label="Toggle menu">
    <!-- hamburger icon -->
  </button>
</nav>
```

To implement a mobile hamburger menu, hide the desktop links with `hidden lg:flex` and show a toggle button with `lg:hidden`. Toggle visibility of a full-width mobile menu panel via JavaScript by adding or removing `hidden`.

To create a dropdown menu, wrap the trigger and dropdown in a `group relative` container. Position the dropdown with `absolute` and animate it in using opacity and visibility transitions:

```html
<div class="relative group">
  <button class="text-sm font-medium text-gray-600 hover:text-gray-900">Products</button>
  <div class="absolute top-full left-0 mt-2 w-48 bg-white rounded-lg shadow-lg border border-gray-200 py-1 opacity-0 invisible group-hover:opacity-100 group-hover:visible transition-all duration-200">
    <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-50">Product A</a>
    <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-50">Product B</a>
  </div>
</div>
```

To indicate the active link, apply a bottom border with `border-b-2 border-blue-600 text-blue-600` for horizontal navs, or a background tint with `bg-blue-50 text-blue-700 rounded-lg` for sidebar navigation.

To build breadcrumbs, lay out items in a flex row with small gaps and muted text:

```html
<nav class="flex items-center gap-2 text-sm text-gray-500">
  <a href="/" class="hover:text-gray-700">Home</a>
  <span>/</span>
  <a href="/products" class="hover:text-gray-700">Products</a>
  <span>/</span>
  <span class="text-gray-900 font-medium">Widget Pro</span>
</nav>
```

## Form Styling

To style a text input, apply full width, rounded corners, a border, padding, and focus ring transitions:

```html
<label class="block text-sm font-medium text-gray-700 mb-1">Email</label>
<input type="email" class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm focus:border-blue-500 focus:ring-1 focus:ring-blue-500 outline-none transition-colors" placeholder="you@example.com" />
```

To style a select dropdown, use `appearance-none` to strip native styling and add a custom chevron via a background image or an adjacent SVG:

```html
<select class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm appearance-none bg-[url('data:image/svg+xml,...')] bg-no-repeat bg-right pr-10 focus:border-blue-500 focus:ring-1 focus:ring-blue-500 outline-none transition-colors">
  <option>Option A</option>
  <option>Option B</option>
</select>
```

To custom-style checkboxes and radios, hide the native input with `peer sr-only` and style the adjacent label based on checked state using `peer-checked:`:

```html
<label class="inline-flex items-center gap-2 cursor-pointer">
  <input type="checkbox" class="peer sr-only" />
  <span class="h-5 w-5 rounded border border-gray-300 bg-white peer-checked:bg-blue-600 peer-checked:border-blue-600 inline-flex items-center justify-center transition-colors">
    <svg class="h-3 w-3 text-white hidden peer-checked:block" viewBox="0 0 12 12"><!-- check icon --></svg>
  </span>
  <span class="text-sm text-gray-700">Accept terms</span>
</label>
```

To show an error state on an input, swap the border and ring colors to red and display an error message below:

```html
<input type="email" class="w-full rounded-lg border border-red-500 px-3 py-2 text-sm focus:border-red-500 focus:ring-1 focus:ring-red-500 outline-none transition-colors" />
<p class="text-red-600 text-sm mt-1">Please enter a valid email address.</p>
```

To maintain consistent spacing across form groups, wrap each label-input pair in a container with `space-y-1` and separate groups with `space-y-4` or `gap-4` in a grid.

## Modal / Dialog

To build a modal overlay, cover the viewport with a fixed, semi-transparent backdrop and center the modal panel:

```html
<div class="fixed inset-0 bg-black/50 z-50 flex items-center justify-center">
  <div class="bg-white rounded-xl shadow-2xl max-w-md w-full mx-4 p-6 relative">
    <button class="absolute top-4 right-4 text-gray-400 hover:text-gray-600" aria-label="Close">
      <svg class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor"><!-- X icon --></svg>
    </button>
    <h2 class="text-lg font-semibold text-gray-900">Modal Title</h2>
    <p class="mt-2 text-gray-600">Modal body content goes here.</p>
    <div class="mt-6 flex justify-end gap-3">
      <button class="px-4 py-2 text-sm font-medium text-gray-700 hover:bg-gray-100 rounded-lg">Cancel</button>
      <button class="px-4 py-2 text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 rounded-lg">Confirm</button>
    </div>
  </div>
</div>
```

To animate the modal entrance, apply `transition-opacity` to the backdrop and `transition-transform` to the panel. Start with `opacity-0` / `scale-95` and transition to `opacity-100` / `scale-100` when the open state is toggled.

To ensure accessibility, prefer the native `<dialog>` element when possible. Apply `aria-modal="true"` and `role="dialog"` to the panel. Implement focus trapping so that Tab and Shift+Tab cycle only through focusable elements inside the modal.

## Table Styling

To make tables responsive, wrap the `<table>` in a container with `overflow-x-auto` so wide tables scroll horizontally on small screens:

```html
<div class="overflow-x-auto rounded-lg border border-gray-200">
  <table class="min-w-full divide-y divide-gray-200">
    <thead class="bg-gray-50">
      <tr>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Name</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Role</th>
      </tr>
    </thead>
    <tbody class="bg-white divide-y divide-gray-200">
      <tr class="even:bg-gray-50 hover:bg-gray-100 transition-colors">
        <td class="px-6 py-4 text-sm text-gray-900 whitespace-nowrap">Jane Cooper</td>
        <td class="px-6 py-4 text-sm text-gray-500 whitespace-nowrap">Active</td>
        <td class="px-6 py-4 text-sm text-gray-500 whitespace-nowrap">Admin</td>
      </tr>
    </tbody>
  </table>
</div>
```

To add striped rows, apply `even:bg-gray-50` to `<tr>` elements. To highlight rows on hover, add `hover:bg-gray-100 transition-colors`. To insert horizontal dividers between rows, apply `divide-y divide-gray-200` to the `<tbody>`.

To create a sticky header that remains visible while scrolling, add `sticky top-0 z-10` to the `<thead>` and ensure the table container has a defined height with `overflow-y-auto`.

## Alerts and Notifications

To display an info alert, use a left border accent with a tinted background:

```html
<div class="bg-blue-50 border-l-4 border-blue-400 p-4 text-blue-700 rounded-r-lg" role="alert">
  <p class="font-medium">Information</p>
  <p class="text-sm mt-1">Your account settings have been updated.</p>
</div>
```

To create success, warning, and error variants, swap the color palette accordingly:
- **Success**: `bg-green-50 border-green-400 text-green-700`
- **Warning**: `bg-yellow-50 border-yellow-400 text-yellow-700`
- **Error**: `bg-red-50 border-red-400 text-red-700`

To make an alert dismissible, add a close button aligned to the top-right corner using `flex justify-between` or absolute positioning.

To build a toast notification, fix it to the bottom-right corner and apply entry animation:

```html
<div class="fixed bottom-4 right-4 z-50 bg-white rounded-lg shadow-lg border border-gray-200 p-4 flex items-center gap-3 animate-[slideUp_0.3s_ease-out]">
  <span class="text-green-500"><!-- check icon --></span>
  <p class="text-sm text-gray-900">Changes saved successfully.</p>
  <button class="text-gray-400 hover:text-gray-600 ml-2" aria-label="Dismiss"><!-- X icon --></button>
</div>
```

## Component Extraction Decision

To decide between `@apply` extraction and framework component abstraction, evaluate the nature of the repeated pattern.

**Use framework components (React/Vue/Svelte)** when:
- The pattern includes logic, state, or interactivity
- Props control variants (primary, secondary, danger)
- The component is framework-specific and benefits from encapsulation

**Use `@apply` extraction** when:
- Styling a base HTML element globally (e.g., `h1`, `a` in prose content)
- Creating a utility shorthand used across many templates without component structure
- Working in a template-only context without a component framework

To implement type-safe variant mapping, use the Class Variance Authority (CVA) library:

```typescript
import { cva } from 'class-variance-authority';

const button = cva('rounded-lg font-medium transition-colors focus-visible:outline-2', {
  variants: {
    intent: {
      primary: 'bg-blue-600 text-white hover:bg-blue-700',
      secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
      danger: 'bg-red-600 text-white hover:bg-red-700',
    },
    size: {
      sm: 'text-sm px-3 py-1.5',
      md: 'text-base px-4 py-2',
      lg: 'text-lg px-6 py-3',
    },
  },
  defaultVariants: {
    intent: 'primary',
    size: 'md',
  },
});
```

To use this in a React component, spread the result of `button({ intent, size })` into the `className` prop. Combine with `tailwind-merge` to allow consumer overrides without class conflicts.

## References

- [Animation Patterns](references/animation-patterns.md) -- deep-dive into transitions, keyframes, enter/leave patterns, and reduced motion handling.
