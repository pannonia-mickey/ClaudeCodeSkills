# Responsive Layout Patterns Reference

## Dashboard Layouts

### Sidebar + Header + Content + Widget Grid

```css
.dashboard {
  display: grid;
  grid-template-rows: auto 1fr;
  grid-template-columns: 1fr;
  min-height: 100dvh;
}

.dashboard__header {
  grid-column: 1 / -1;
  position: sticky;
  top: 0;
  z-index: var(--z-header);
  background: var(--color-bg-elevated);
  border-block-end: 1px solid var(--color-border-default);
  padding: var(--space-3) var(--space-4);
}

.dashboard__sidebar {
  display: none; /* Hidden on mobile */
}

.dashboard__content {
  padding: var(--space-4);
  overflow-y: auto;
}

@media (min-width: 1024px) {
  .dashboard {
    grid-template-columns: 260px 1fr;
  }

  .dashboard__sidebar {
    display: flex;
    flex-direction: column;
    border-inline-end: 1px solid var(--color-border-default);
    padding: var(--space-4);
    overflow-y: auto;
    height: calc(100dvh - var(--header-height));
    position: sticky;
    top: var(--header-height);
  }
}
```

### Collapsible Sidebar

```css
.sidebar {
  width: var(--sidebar-width, 260px);
  transition: width var(--duration-normal) var(--ease-default);
  overflow: hidden;
}

.sidebar--collapsed {
  --sidebar-width: 64px;
}

.sidebar--collapsed .sidebar__label {
  opacity: 0;
  width: 0;
  overflow: hidden;
}

.sidebar__toggle {
  position: absolute;
  inset-inline-end: -12px;
  top: var(--space-4);
}
```

### Widget Grid (Dashboard Tiles)

```css
.widget-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(min(320px, 100%), 1fr));
  gap: var(--space-4);
}

/* Span-2 widget for wide charts */
.widget--wide {
  grid-column: span 2;
}

/* Full-width widget */
.widget--full {
  grid-column: 1 / -1;
}

/* On mobile, all widgets are full-width */
@media (max-width: 639px) {
  .widget--wide {
    grid-column: span 1;
  }
}
```

---

## Split-View Patterns

### Master-Detail

```css
.master-detail {
  display: grid;
  grid-template-columns: 1fr;
  min-height: 100%;
}

/* Mobile: list and detail are full-width, toggled via state */
.master-detail--showing-detail .master-list {
  display: none;
}

.master-detail--showing-list .detail-panel {
  display: none;
}

@media (min-width: 768px) {
  .master-detail {
    grid-template-columns: 350px 1fr;
  }

  /* Both always visible on tablet+ */
  .master-detail--showing-detail .master-list,
  .master-detail--showing-list .detail-panel {
    display: block;
  }
}

@media (min-width: 1280px) {
  .master-detail {
    grid-template-columns: 400px 1fr;
  }
}
```

### Comparison / Side-by-Side

```css
.comparison {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--space-4);
}

@media (min-width: 768px) {
  .comparison {
    grid-template-columns: 1fr 1fr;
  }

  .comparison__divider {
    display: block;
    width: 1px;
    background: var(--color-border-default);
  }
}

/* Synchronized scrolling via JS */
.comparison__panel {
  overflow-y: auto;
  max-height: calc(100dvh - var(--header-height));
}
```

### Side-by-Side Editing (Code + Preview)

```css
.editor-preview {
  display: grid;
  grid-template-columns: 1fr;
  grid-template-rows: 1fr 1fr;
  height: calc(100dvh - var(--header-height));
}

@media (min-width: 1024px) {
  .editor-preview {
    grid-template-columns: 1fr 1fr;
    grid-template-rows: 1fr;
  }
}

.editor-pane {
  overflow: auto;
  border-inline-end: 1px solid var(--color-border-default);
}

.preview-pane {
  overflow: auto;
  background: var(--color-bg-secondary);
}
```

---

## Masonry Grids

### CSS Columns Approach (Simple)

```css
.masonry {
  column-count: 1;
  column-gap: var(--space-4);
}

.masonry__item {
  break-inside: avoid;
  margin-block-end: var(--space-4);
}

@media (min-width: 640px) {
  .masonry { column-count: 2; }
}

@media (min-width: 1024px) {
  .masonry { column-count: 3; }
}

@media (min-width: 1280px) {
  .masonry { column-count: 4; }
}
```

**Limitation:** Items flow top-to-bottom in columns, not left-to-right in rows. Reading order may not match visual order.

### CSS Grid with `grid-template-rows: masonry` (Experimental)

```css
/* Firefox behind flag, Chrome under development */
.masonry-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(min(280px, 100%), 1fr));
  grid-template-rows: masonry;
  gap: var(--space-4);
}
```

### JavaScript Fallback for True Masonry

```javascript
function layoutMasonry(container, columnWidth) {
  const items = [...container.children];
  const containerWidth = container.offsetWidth;
  const columns = Math.floor(containerWidth / columnWidth);
  const columnHeights = new Array(columns).fill(0);

  items.forEach(item => {
    const shortestColumn = columnHeights.indexOf(Math.min(...columnHeights));
    item.style.position = 'absolute';
    item.style.left = `${shortestColumn * columnWidth}px`;
    item.style.top = `${columnHeights[shortestColumn]}px`;
    item.style.width = `${columnWidth}px`;
    columnHeights[shortestColumn] += item.offsetHeight + gap;
  });

  container.style.height = `${Math.max(...columnHeights)}px`;
  container.style.position = 'relative';
}
```

---

## Responsive Navigation Patterns

### Mega Menu

```css
.mega-menu {
  display: none;
  position: absolute;
  inset-inline: 0;
  top: 100%;
  background: var(--color-bg-elevated);
  box-shadow: var(--shadow-lg);
  padding: var(--space-6);
}

.nav-item:hover .mega-menu,
.nav-item:focus-within .mega-menu {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: var(--space-6);
}

/* On mobile, mega menu becomes accordion */
@media (max-width: 1023px) {
  .mega-menu {
    position: static;
    box-shadow: none;
    padding: var(--space-2) var(--space-4);
  }

  .nav-item:hover .mega-menu,
  .nav-item:focus-within .mega-menu {
    display: block;
  }
}
```

### Off-Canvas Navigation

```css
.off-canvas {
  position: fixed;
  inset-block: 0;
  inset-inline-start: 0;
  width: min(320px, 85vw);
  transform: translateX(-100%);
  transition: transform var(--duration-normal) var(--ease-default);
  background: var(--color-bg-primary);
  z-index: var(--z-drawer);
  overflow-y: auto;
}

.off-canvas--open {
  transform: translateX(0);
}

/* Backdrop */
.off-canvas-backdrop {
  position: fixed;
  inset: 0;
  background: var(--color-bg-overlay);
  opacity: 0;
  pointer-events: none;
  transition: opacity var(--duration-normal);
}

.off-canvas--open ~ .off-canvas-backdrop {
  opacity: 1;
  pointer-events: auto;
}

/* On desktop, no off-canvas needed */
@media (min-width: 1024px) {
  .off-canvas {
    position: static;
    transform: none;
    width: auto;
  }
}
```

### Bottom Sheet Navigation (Mobile)

```css
.bottom-sheet {
  position: fixed;
  inset-inline: 0;
  bottom: 0;
  background: var(--color-bg-elevated);
  border-radius: var(--radius-xl) var(--radius-xl) 0 0;
  box-shadow: var(--shadow-xl);
  transform: translateY(100%);
  transition: transform var(--duration-normal) var(--ease-default);
  max-height: 85dvh;
  overflow-y: auto;
  z-index: var(--z-drawer);
}

.bottom-sheet--open {
  transform: translateY(0);
}

.bottom-sheet__handle {
  width: 40px;
  height: 4px;
  background: var(--color-border-default);
  border-radius: var(--radius-full);
  margin: var(--space-2) auto var(--space-4);
}

/* Not needed on desktop */
@media (min-width: 1024px) {
  .bottom-sheet {
    display: none;
  }
}
```

---

## Data-Dense Layouts

### Table to Card Transformation

```css
/* Desktop: standard table */
.data-table {
  width: 100%;
  border-collapse: collapse;
}

.data-table th,
.data-table td {
  padding: var(--space-2) var(--space-3);
  text-align: start;
  border-block-end: 1px solid var(--color-border-default);
}

/* Mobile: card layout */
@media (max-width: 767px) {
  .data-table thead {
    display: none; /* Hide column headers */
  }

  .data-table tr {
    display: block;
    padding: var(--space-3);
    margin-block-end: var(--space-3);
    border: 1px solid var(--color-border-default);
    border-radius: var(--radius-md);
  }

  .data-table td {
    display: flex;
    justify-content: space-between;
    padding: var(--space-1) 0;
    border: none;
  }

  .data-table td::before {
    content: attr(data-label);
    font-weight: var(--font-weight-semibold);
    margin-inline-end: var(--space-4);
  }
}
```

### Horizontal Scroll Table (Alternative)

```css
.table-scroll-wrapper {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
  border: 1px solid var(--color-border-default);
  border-radius: var(--radius-md);
}

/* Scroll shadow indicators */
.table-scroll-wrapper {
  background:
    linear-gradient(to right, var(--color-bg-primary) 30%, transparent),
    linear-gradient(to left, var(--color-bg-primary) 30%, transparent),
    linear-gradient(to right, rgba(0,0,0,0.1), transparent),
    linear-gradient(to left, rgba(0,0,0,0.1), transparent);
  background-position: left center, right center, left center, right center;
  background-repeat: no-repeat;
  background-size: 40px 100%, 40px 100%, 14px 100%, 14px 100%;
  background-attachment: local, local, scroll, scroll;
}
```

### Column Hiding Strategy

```css
/* Priority-based column hiding */
.data-table .col-priority-1 { /* Always visible */ }
.data-table .col-priority-2 { display: none; }
.data-table .col-priority-3 { display: none; }

@media (min-width: 768px) {
  .data-table .col-priority-2 { display: table-cell; }
}

@media (min-width: 1280px) {
  .data-table .col-priority-3 { display: table-cell; }
}
```

---

## Multi-Panel Layouts

### Resizable Panels

```css
.resizable-layout {
  display: grid;
  grid-template-columns: var(--panel-left, 1fr) 4px var(--panel-right, 1fr);
  height: calc(100dvh - var(--header-height));
}

.resize-handle {
  cursor: col-resize;
  background: var(--color-border-default);
  transition: background var(--duration-fast);
}

.resize-handle:hover,
.resize-handle:active {
  background: var(--color-primary);
}
```

```javascript
// Resize handler
const handle = document.querySelector('.resize-handle');
const layout = document.querySelector('.resizable-layout');

handle.addEventListener('mousedown', (e) => {
  const startX = e.clientX;
  const startWidth = layout.querySelector('.panel-left').offsetWidth;

  function onMouseMove(e) {
    const newWidth = startWidth + (e.clientX - startX);
    const minWidth = 200;
    const maxWidth = layout.offsetWidth - 200;
    const clampedWidth = Math.max(minWidth, Math.min(maxWidth, newWidth));
    layout.style.setProperty('--panel-left', `${clampedWidth}px`);
  }

  function onMouseUp() {
    document.removeEventListener('mousemove', onMouseMove);
    document.removeEventListener('mouseup', onMouseUp);
  }

  document.addEventListener('mousemove', onMouseMove);
  document.addEventListener('mouseup', onMouseUp);
});
```

---

## Full-Page App Layouts

### Fixed Header + Sticky Sidebar + Scrollable Content

```css
.app-shell {
  display: grid;
  grid-template-rows: var(--header-height) 1fr;
  grid-template-columns: 1fr;
  height: 100dvh;
  overflow: hidden;
}

.app-header {
  grid-column: 1 / -1;
  border-block-end: 1px solid var(--color-border-default);
}

.app-body {
  display: grid;
  grid-template-columns: 1fr;
  overflow: hidden;
}

@media (min-width: 1024px) {
  .app-body {
    grid-template-columns: var(--sidebar-width) 1fr;
  }
}

.app-sidebar {
  overflow-y: auto;
  border-inline-end: 1px solid var(--color-border-default);
}

.app-main {
  overflow-y: auto;
  padding: var(--space-6);
}
```

### Sticky Elements Within Scrollable Content

```css
.content-header {
  position: sticky;
  top: 0;
  z-index: var(--z-sticky);
  background: var(--color-bg-primary);
  border-block-end: 1px solid var(--color-border-default);
  padding: var(--space-3) var(--space-4);
}

/* Sticky sidebar within a scrollable section */
.sticky-aside {
  position: sticky;
  top: var(--space-4);
  align-self: start;
}
```
