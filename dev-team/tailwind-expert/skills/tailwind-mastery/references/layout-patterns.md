# Layout Patterns

This reference provides a deep-dive into layout patterns using Tailwind CSS, covering Flexbox and Grid decision-making, common layout recipes with complete class lists, container queries, and positioning patterns.

## Flexbox vs Grid Decision Tree

To choose between Flexbox and Grid, evaluate the layout requirements against each model's strengths.

### When to Use Flexbox

To handle one-dimensional layouts, reach for Flexbox. Use it when:

1. **Content drives sizing.** To let items size themselves based on their content and distribute remaining space, Flexbox excels. A navigation bar where each link has a different text length but all sit in a single row is a natural Flexbox use case. Apply `flex items-center gap-6` to the container and let each nav item take its natural width.

2. **Wrapping behavior is needed.** To allow items to flow onto the next line when space runs out, use `flex flex-wrap gap-4`. Flexbox wrapping is ideal for tag lists, button groups, or any collection where the number of items varies and they should reflow naturally.

3. **Alignment within a single axis is the primary concern.** To vertically center an icon next to text, use `flex items-center gap-2`. To push items to opposite ends of a row, use `flex items-center justify-between`. These single-axis alignment tasks are what Flexbox was designed for.

4. **Items need to grow or shrink proportionally.** To make one item fill remaining space while others stay fixed, apply `flex-1` to the growing item and `flex-shrink-0` to fixed items. A search bar that expands to fill available space between a logo and action buttons is a classic example.

### When to Use Grid

To handle two-dimensional layouts or precise placement, reach for Grid. Use it when:

1. **The layout has rows AND columns.** To create a dashboard with cards arranged in both dimensions, Grid provides direct control over both axes simultaneously. Apply `grid grid-cols-3 gap-6` and let Grid handle the two-dimensional arrangement.

2. **Precise placement is required.** To place an item in a specific row and column, use `col-start-2 row-start-1`. Grid allows explicit placement that Flexbox cannot match without awkward workarounds.

3. **Overlapping elements are needed.** To layer elements on top of each other within a grid, assign them to the same grid area. This is useful for image overlays, stacked card effects, or any design where elements intentionally overlap.

4. **Equal-height columns are required without tricks.** To make all items in a row the same height regardless of content length, Grid handles this natively. Apply `grid grid-cols-3` and every item in the same row stretches to the tallest item's height. Flexbox can achieve this with `items-stretch`, but Grid's behavior is more predictable in complex layouts.

5. **The layout structure is defined by the container, not the content.** To impose a rigid layout structure regardless of what content appears, Grid's template-based approach is the right tool. Define the tracks on the container and place content into them.

## Common Layout Recipes

### 1. Sidebar Layout

To create a fixed sidebar with scrollable main content, use CSS Grid with explicit column tracks:

```html
<div class="grid grid-cols-[16rem_1fr] h-screen">
  <aside class="overflow-y-auto border-r border-gray-200 bg-gray-50 p-4">
    <!-- Sidebar navigation -->
  </aside>
  <main class="overflow-y-auto p-6">
    <!-- Main content -->
  </main>
</div>
```

To make the sidebar collapsible on mobile, add responsive classes: `grid grid-cols-1 md:grid-cols-[16rem_1fr]` and conditionally hide the sidebar with `hidden md:block` or use a slide-out drawer pattern on small screens.

### 2. Holy Grail Layout

To build the classic header, sidebar, content, aside, and footer layout, use Grid template areas:

```html
<div class="grid min-h-screen grid-rows-[auto_1fr_auto] grid-cols-[16rem_1fr_12rem]">
  <header class="col-span-3 border-b border-gray-200 bg-white px-6 py-4">
    <!-- Header -->
  </header>
  <nav class="overflow-y-auto border-r border-gray-200 bg-gray-50 p-4">
    <!-- Left sidebar -->
  </nav>
  <main class="overflow-y-auto p-6">
    <!-- Main content -->
  </main>
  <aside class="overflow-y-auto border-l border-gray-200 bg-gray-50 p-4">
    <!-- Right aside -->
  </aside>
  <footer class="col-span-3 border-t border-gray-200 bg-white px-6 py-4">
    <!-- Footer -->
  </footer>
</div>
```

To make this responsive, collapse the sidebars on smaller screens: use `grid-cols-1 lg:grid-cols-[16rem_1fr_12rem]` and `hidden lg:block` on the sidebars.

### 3. Sticky Footer Layout

To ensure the footer always sits at the bottom of the viewport even when content is short, use Flexbox with a growing main area:

```html
<div class="flex min-h-screen flex-col">
  <header class="border-b border-gray-200 bg-white px-6 py-4">
    <!-- Header -->
  </header>
  <main class="flex-1 p-6">
    <!-- Main content: flex-1 pushes footer down -->
  </main>
  <footer class="border-t border-gray-200 bg-gray-100 px-6 py-4">
    <!-- Footer -->
  </footer>
</div>
```

The `min-h-screen` on the container ensures it fills the viewport. The `flex-1` on `<main>` forces it to absorb all remaining vertical space, pushing the footer to the bottom regardless of content height.

### 4. Centered Content Layout

To center page content with a maximum width and responsive horizontal padding, use the standard centering pattern:

```html
<div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
  <!-- Centered content with responsive padding -->
</div>
```

To nest narrower content sections within the centered container, apply additional `max-w-` constraints: `max-w-3xl mx-auto` for prose-width text, `max-w-lg mx-auto` for forms.

To create a full-bleed section within centered content, break out of the container with negative margins and padding:

```html
<div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
  <p>Normal centered content.</p>
  <div class="-mx-4 bg-blue-50 px-4 py-12 sm:-mx-6 sm:px-6 lg:-mx-8 lg:px-8">
    <!-- Full-bleed background within centered container -->
  </div>
</div>
```

### 5. Responsive Card Grid

To build a card grid that adapts from one column on mobile to multiple columns on larger screens, use Grid with responsive column counts:

```html
<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
  <article class="overflow-hidden rounded-xl bg-white shadow-md">
    <img src="image.jpg" alt="" class="h-48 w-full object-cover" />
    <div class="p-6">
      <h3 class="text-lg font-semibold text-gray-900">Card Title</h3>
      <p class="mt-2 text-sm text-gray-600">Card description text goes here.</p>
    </div>
  </article>
  <!-- More cards... -->
</div>
```

To add a fourth column on extra-large screens, append `xl:grid-cols-4`. To ensure cards have equal height, Grid handles this automatically since all items in a row share the same implicit row height.

### 6. Split Layout (50/50 Hero)

To create a half-image, half-content hero section, use Grid with two equal columns:

```html
<section class="grid min-h-[60vh] grid-cols-1 lg:grid-cols-2">
  <div class="flex flex-col justify-center px-6 py-12 lg:px-16">
    <h1 class="text-4xl font-bold tracking-tight text-gray-900">
      Headline text here
    </h1>
    <p class="mt-4 text-lg text-gray-600">
      Supporting paragraph text with details.
    </p>
    <div class="mt-8 flex gap-4">
      <a href="#" class="rounded-lg bg-blue-600 px-6 py-3 text-sm font-semibold text-white shadow-sm hover:bg-blue-500">
        Primary CTA
      </a>
      <a href="#" class="rounded-lg border border-gray-300 px-6 py-3 text-sm font-semibold text-gray-700 hover:bg-gray-50">
        Secondary CTA
      </a>
    </div>
  </div>
  <div class="relative">
    <img src="hero.jpg" alt="" class="absolute inset-0 h-full w-full object-cover" />
  </div>
</section>
```

To reverse the order on mobile (image first), use `order-first lg:order-none` on the image container. To stack vertically on mobile, the `grid-cols-1 lg:grid-cols-2` already handles this.

### 7. Masonry-Like Layout

To approximate a masonry layout using CSS columns, use the `columns` utility:

```html
<div class="columns-1 gap-4 space-y-4 sm:columns-2 md:columns-3">
  <div class="break-inside-avoid rounded-lg bg-white p-4 shadow">
    <p>Short content.</p>
  </div>
  <div class="break-inside-avoid rounded-lg bg-white p-4 shadow">
    <p>Longer content that takes up more vertical space to demonstrate the masonry effect.</p>
  </div>
  <div class="break-inside-avoid rounded-lg bg-white p-4 shadow">
    <img src="photo.jpg" alt="" class="w-full rounded" />
    <p class="mt-2">Caption text.</p>
  </div>
  <!-- More items... -->
</div>
```

The `break-inside-avoid` class prevents items from splitting across columns. The `space-y-4` adds vertical spacing between items within each column. Note that items flow top-to-bottom then left-to-right (column-first order), unlike a true masonry layout which fills rows. For true masonry behavior, consider JavaScript-based solutions or wait for the CSS `masonry` value for `grid-template-rows` to gain wider browser support.

## Container Queries

To build components that adapt to their container's width rather than the viewport, use container queries. This is essential for reusable components that appear in different layout contexts (a card in a sidebar vs. in a main content area).

### Setting Up Container Queries

To establish a containment context, apply `@container` on the parent element. To query against that container, use size-based variants on the children:

```html
<div class="@container">
  <div class="flex flex-col @sm:flex-row @sm:items-center gap-4">
    <img src="avatar.jpg" alt="" class="size-16 rounded-full @sm:size-12" />
    <div>
      <h3 class="text-lg font-semibold @sm:text-base">User Name</h3>
      <p class="text-sm text-gray-500">Role description</p>
    </div>
  </div>
</div>
```

The `@sm:` prefix applies when the container is at least 20rem (320px) wide. The available container breakpoints mirror the standard breakpoints: `@xs` (16rem), `@sm` (20rem), `@md` (28rem), `@lg` (32rem), `@xl` (36rem), `@2xl` (42rem), and larger.

### Named Containers

To disambiguate nested containers, use named containers. Apply `@container/sidebar` on the parent and `@sm/sidebar:` on children to query specifically against the sidebar container:

```html
<div class="@container/sidebar">
  <div class="@container/card">
    <p class="@md/sidebar:text-lg @sm/card:font-bold">
      Responds to different containers.
    </p>
  </div>
</div>
```

### Use Cases for Container Queries

To identify where container queries provide the most value, consider these scenarios:

- **Sidebar widgets** that switch from a horizontal to vertical layout depending on whether the sidebar is expanded or collapsed.
- **Dashboard cards** that show more or less detail based on the grid cell size.
- **Reusable component libraries** where the component author cannot know the viewport context in advance.
- **Responsive navigation** within a panel that adapts to the panel's width, not the page width.

### Tailwind v4 Container Query Syntax

To use container queries in Tailwind v4, note that native support is built in. The `@container` utility and `@sm:`, `@md:`, etc. variants work out of the box without additional plugins. In v3, the `@tailwindcss/container-queries` plugin was required.

## Positioning Patterns

### Sticky Header

To create a header that sticks to the top of the viewport during scroll, use `sticky` positioning with a backdrop blur for a polished effect:

```html
<header class="sticky top-0 z-40 border-b border-gray-200/50 bg-white/80 backdrop-blur-lg">
  <div class="mx-auto flex max-w-7xl items-center justify-between px-4 py-3 sm:px-6 lg:px-8">
    <a href="/" class="text-lg font-bold">Logo</a>
    <nav class="flex items-center gap-6">
      <a href="#" class="text-sm font-medium text-gray-700 hover:text-gray-900">Link</a>
    </nav>
  </div>
</header>
```

The `bg-white/80` with `backdrop-blur-lg` creates a frosted glass effect. The `z-40` ensures the header sits above page content but below modals (typically `z-50`).

### Full-Screen Overlay

To create a modal backdrop or full-screen overlay, use fixed positioning covering the entire viewport:

```html
<div class="fixed inset-0 z-50 bg-black/50 backdrop-blur-sm">
  <div class="flex min-h-full items-center justify-center p-4">
    <div class="w-full max-w-md rounded-xl bg-white p-6 shadow-xl">
      <!-- Modal content -->
    </div>
  </div>
</div>
```

The `inset-0` shorthand sets `top: 0; right: 0; bottom: 0; left: 0`, stretching the overlay across the viewport. The `bg-black/50` applies a semi-transparent black background.

### Absolute Positioning Within Relative Parent

To position an element relative to its parent, set the parent to `relative` and the child to `absolute`:

```html
<div class="relative">
  <img src="product.jpg" alt="" class="h-64 w-full rounded-lg object-cover" />
  <span class="absolute right-2 top-2 rounded-full bg-red-500 px-2 py-0.5 text-xs font-bold text-white">
    SALE
  </span>
  <div class="absolute inset-x-0 bottom-0 rounded-b-lg bg-gradient-to-t from-black/60 to-transparent p-4">
    <h3 class="text-lg font-semibold text-white">Product Name</h3>
  </div>
</div>
```

The badge sits in the top-right corner, and the overlay with the product name anchors to the bottom. Both are positioned relative to the image's container.

### Floating Action Button

To create a floating action button fixed in the bottom-right corner, use fixed positioning:

```html
<button class="fixed bottom-6 right-6 z-40 flex size-14 items-center justify-center rounded-full bg-blue-600 text-white shadow-lg transition-transform hover:scale-110 hover:bg-blue-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600 active:scale-95">
  <svg class="size-6" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor">
    <path stroke-linecap="round" stroke-linejoin="round" d="M12 4.5v15m7.5-7.5h-15" />
  </svg>
</button>
```

The `fixed bottom-6 right-6` anchors the button 1.5rem from the bottom-right corner of the viewport. The `hover:scale-110 active:scale-95` adds a satisfying interaction feel.

### Tooltip with Absolute Positioning

To create a tooltip that appears above an element on hover, combine relative/absolute positioning with group hover:

```html
<div class="group relative inline-block">
  <button class="text-sm text-gray-600 underline decoration-dotted">
    Hover me
  </button>
  <div class="invisible absolute bottom-full left-1/2 mb-2 -translate-x-1/2 rounded-lg bg-gray-900 px-3 py-1.5 text-xs text-white opacity-0 shadow-lg transition-all group-hover:visible group-hover:opacity-100">
    Tooltip content here
    <div class="absolute left-1/2 top-full -translate-x-1/2 border-4 border-transparent border-t-gray-900"></div>
  </div>
</div>
```

The tooltip is invisible by default and fades in on group hover. The small triangle at the bottom uses CSS border tricks positioned absolutely below the tooltip body.
