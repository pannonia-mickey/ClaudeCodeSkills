# ARIA Widget Patterns Reference

## Menu and Menubar

### Structure

```html
<nav aria-label="Main">
  <ul role="menubar" aria-label="Application menu">
    <li role="none">
      <button role="menuitem" aria-haspopup="true" aria-expanded="false">
        File
      </button>
      <ul role="menu" aria-label="File">
        <li role="none">
          <button role="menuitem">New</button>
        </li>
        <li role="none">
          <button role="menuitem">Open</button>
        </li>
        <li role="separator"></li>
        <li role="none">
          <button role="menuitem" aria-haspopup="true" aria-expanded="false">
            Export
          </button>
          <ul role="menu" aria-label="Export formats">
            <li role="none"><button role="menuitem">PDF</button></li>
            <li role="none"><button role="menuitem">CSV</button></li>
          </ul>
        </li>
      </ul>
    </li>
  </ul>
</nav>
```

### Keyboard Interaction

| Key | Menubar | Menu (Dropdown) |
|-----|---------|----------------|
| `Enter` / `Space` | Open submenu or activate item | Activate item |
| `Arrow Right` | Next menubar item | Open submenu (if has submenu) |
| `Arrow Left` | Previous menubar item | Close submenu, return to parent |
| `Arrow Down` | Open submenu | Next menu item |
| `Arrow Up` | — | Previous menu item |
| `Home` | First menubar item | First menu item |
| `End` | Last menubar item | Last menu item |
| `Escape` | — | Close menu, return focus to trigger |
| Type character | Jump to matching item | Jump to matching item |

### Focus Management
- Menubar uses roving tabindex: active item has `tabindex="0"`, others `tabindex="-1"`
- When a menu opens, focus moves to the first menu item
- When a menu closes, focus returns to its trigger button

---

## Tree View

### Structure

```html
<ul role="tree" aria-label="File browser">
  <li role="treeitem" aria-expanded="true" aria-selected="false">
    <span class="tree-label">src/</span>
    <ul role="group">
      <li role="treeitem" aria-expanded="false" aria-selected="false">
        <span class="tree-label">components/</span>
        <ul role="group">
          <li role="treeitem" aria-selected="true" class="tree-leaf">
            Button.tsx
          </li>
          <li role="treeitem" aria-selected="false" class="tree-leaf">
            Card.tsx
          </li>
        </ul>
      </li>
      <li role="treeitem" aria-selected="false" class="tree-leaf">
        index.ts
      </li>
    </ul>
  </li>
</ul>
```

### Keyboard Interaction

| Key | Action |
|-----|--------|
| `Arrow Down` | Next visible treeitem |
| `Arrow Up` | Previous visible treeitem |
| `Arrow Right` | Expand closed node; or move to first child; or do nothing on leaf |
| `Arrow Left` | Collapse open node; or move to parent |
| `Home` | First treeitem |
| `End` | Last visible treeitem |
| `Enter` | Activate/select item |
| `Space` | Toggle selection (multi-select mode) |
| `*` | Expand all siblings at current level |
| Type character | Jump to next item starting with that character |

### Multi-Select Tree

For multi-select, add `aria-multiselectable="true"` to the `tree` element:
- `Space` toggles selection on focused item
- `Shift + Arrow` extends selection
- `Ctrl + Space` toggles without deselecting others
- `Ctrl + A` selects all

---

## Data Grid

### Structure

```html
<div role="grid" aria-label="Employees" aria-rowcount="150">
  <div role="rowgroup">
    <div role="row">
      <div role="columnheader" aria-sort="ascending">
        Name
        <button aria-label="Sort by name">↑</button>
      </div>
      <div role="columnheader" aria-sort="none">Email</div>
      <div role="columnheader" aria-sort="none">Role</div>
    </div>
  </div>
  <div role="rowgroup">
    <div role="row" aria-rowindex="1" aria-selected="false">
      <div role="gridcell">Alice Johnson</div>
      <div role="gridcell">alice@example.com</div>
      <div role="gridcell">Engineer</div>
    </div>
    <div role="row" aria-rowindex="2" aria-selected="true">
      <div role="gridcell">Bob Smith</div>
      <div role="gridcell">bob@example.com</div>
      <div role="gridcell">Designer</div>
    </div>
  </div>
</div>
```

### Keyboard Interaction

| Key | Action |
|-----|--------|
| `Arrow Right` | Move to next cell in row |
| `Arrow Left` | Move to previous cell in row |
| `Arrow Down` | Move to same column in next row |
| `Arrow Up` | Move to same column in previous row |
| `Home` | First cell in row |
| `End` | Last cell in row |
| `Ctrl + Home` | First cell in grid |
| `Ctrl + End` | Last cell in grid |
| `Page Down` | Scroll down by visible rows |
| `Page Up` | Scroll up by visible rows |
| `Space` | Select/deselect row (if selectable) |
| `Enter` | Enter edit mode on cell / activate link |
| `Escape` | Exit edit mode, discard changes |
| `F2` | Enter edit mode on cell |
| `Tab` | Move to next interactive element outside grid |

### Sort Announcements

When sort changes, update `aria-sort` and announce:
```javascript
function updateSort(columnHeader, direction) {
  // Remove sort from all columns
  document.querySelectorAll('[role="columnheader"]')
    .forEach(h => h.setAttribute('aria-sort', 'none'));

  // Set new sort
  columnHeader.setAttribute('aria-sort', direction);

  // Announce change
  liveRegion.textContent = `Sorted by ${columnHeader.textContent}, ${direction}`;
}
```

---

## Toolbar

### Structure

```html
<div role="toolbar" aria-label="Text formatting" aria-orientation="horizontal">
  <button aria-pressed="false" aria-label="Bold" tabindex="0">B</button>
  <button aria-pressed="false" aria-label="Italic" tabindex="-1">I</button>
  <button aria-pressed="false" aria-label="Underline" tabindex="-1">U</button>

  <div role="separator" aria-orientation="vertical"></div>

  <button aria-label="Align left" tabindex="-1" aria-pressed="true">⫷</button>
  <button aria-label="Align center" tabindex="-1" aria-pressed="false">⫸</button>

  <div role="separator" aria-orientation="vertical"></div>

  <select aria-label="Font size" tabindex="-1">
    <option>12</option>
    <option selected>14</option>
    <option>16</option>
  </select>
</div>
```

### Keyboard Interaction

| Key | Action |
|-----|--------|
| `Arrow Right` | Next toolbar item (roving tabindex) |
| `Arrow Left` | Previous toolbar item |
| `Home` | First toolbar item |
| `End` | Last toolbar item |
| `Tab` | Leave toolbar to next focusable element outside |
| `Shift + Tab` | Leave toolbar to previous focusable element outside |
| `Enter` / `Space` | Activate button, toggle pressed state |

The toolbar acts as a single tab stop. Internal navigation uses arrow keys with roving tabindex.

---

## Carousel

### Structure

```html
<section aria-roledescription="carousel" aria-label="Featured products">
  <div class="carousel-controls">
    <button aria-label="Previous slide" class="carousel-prev">←</button>
    <button aria-label="Next slide" class="carousel-next">→</button>
    <button aria-label="Pause auto-rotation" aria-pressed="false" class="carousel-pause">⏸</button>
  </div>

  <div aria-live="off" class="carousel-slides">
    <div role="group" aria-roledescription="slide" aria-label="1 of 5">
      <img src="product1.jpg" alt="Wireless headphones - $99" />
      <a href="/products/1">View details</a>
    </div>
    <div role="group" aria-roledescription="slide" aria-label="2 of 5" hidden>
      <img src="product2.jpg" alt="Smart watch - $249" />
      <a href="/products/2">View details</a>
    </div>
  </div>

  <!-- Slide indicators -->
  <div role="tablist" aria-label="Slides">
    <button role="tab" aria-selected="true" aria-label="Slide 1">●</button>
    <button role="tab" aria-selected="false" aria-label="Slide 2">○</button>
  </div>
</section>
```

### Auto-Rotation Rules

- Must have a visible pause button
- Auto-rotation pauses on hover and focus
- `aria-live` on slide container:
  - `"off"` when auto-rotating (to prevent constant announcements)
  - `"polite"` when user is manually navigating
- Rotation stops permanently when user interacts with any slide control

---

## Feed (Infinite Scroll)

### Structure

```html
<div role="feed" aria-label="News articles" aria-busy="false">
  <article aria-posinset="1" aria-setsize="-1" tabindex="0" aria-labelledby="article-1-title">
    <h2 id="article-1-title">Article Title</h2>
    <p>Article excerpt...</p>
  </article>
  <article aria-posinset="2" aria-setsize="-1" tabindex="0" aria-labelledby="article-2-title">
    <h2 id="article-2-title">Another Article</h2>
    <p>Article excerpt...</p>
  </article>
</div>
```

### Key Attributes

- `aria-setsize="-1"`: Total count unknown (infinite)
- `aria-posinset`: Position of each article in the feed
- `aria-busy="true"`: Set while loading more items
- Each article must be focusable and labeled

### Keyboard Interaction

| Key | Action |
|-----|--------|
| `Page Down` | Next article |
| `Page Up` | Previous article |
| `Ctrl + End` | Move past the feed |

---

## Disclosure / Accordion

### Single Disclosure

```html
<div class="disclosure">
  <button aria-expanded="false" aria-controls="panel-1" class="disclosure__trigger">
    More details
  </button>
  <div id="panel-1" class="disclosure__panel" hidden>
    <p>Expanded content here.</p>
  </div>
</div>
```

### Accordion (Grouped Disclosures)

```html
<div class="accordion" role="region" aria-label="FAQ">
  <h3>
    <button aria-expanded="true" aria-controls="faq-1" id="faq-1-trigger">
      How do I reset my password?
    </button>
  </h3>
  <div id="faq-1" role="region" aria-labelledby="faq-1-trigger">
    <p>Visit the login page and click "Forgot password"...</p>
  </div>

  <h3>
    <button aria-expanded="false" aria-controls="faq-2" id="faq-2-trigger">
      How do I contact support?
    </button>
  </h3>
  <div id="faq-2" role="region" aria-labelledby="faq-2-trigger" hidden>
    <p>Email support@example.com or use the in-app chat...</p>
  </div>
</div>
```

**UX Decision:** Allow multiple panels open simultaneously unless there's a strong reason for mutual exclusivity. Mutual exclusivity can frustrate users who want to compare content.

---

## Toast / Notification System

### Structure

```html
<!-- Toast container — fixed position -->
<div class="toast-container" aria-live="polite" aria-atomic="false" aria-relevant="additions">
  <div class="toast toast--success" role="status">
    <svg aria-hidden="true"><!-- check icon --></svg>
    <p class="toast__message">Project saved successfully</p>
    <button aria-label="Dismiss notification" class="toast__close">×</button>
  </div>

  <!-- Error toasts use role="alert" for immediate announcement -->
  <div class="toast toast--error" role="alert">
    <svg aria-hidden="true"><!-- error icon --></svg>
    <p class="toast__message">Failed to save project</p>
    <button class="toast__action">Retry</button>
    <button aria-label="Dismiss notification" class="toast__close">×</button>
  </div>
</div>
```

### Toast Rules

| Type | `role` | Auto-dismiss | Has Action |
|------|--------|-------------|------------|
| Success | `status` | Yes (5s) | No |
| Info | `status` | Yes (8s) | Optional |
| Warning | `status` | No | Optional |
| Error | `alert` | No | Yes (retry) |

- Success/info toasts auto-dismiss; warning/error toasts require user action
- Toasts must be keyboard-dismissable
- Auto-dismiss pauses on hover and keyboard focus
- Stack new toasts, don't replace existing ones
- Maximum 3 toasts visible simultaneously; queue the rest

---

## Command Palette / Search Dialog

### Structure

```html
<div role="dialog" aria-modal="true" aria-label="Command palette">
  <div role="combobox" aria-expanded="true" aria-haspopup="listbox" aria-owns="command-list">
    <input type="text" aria-autocomplete="list" aria-controls="command-list"
           aria-activedescendant="cmd-3" placeholder="Type a command..." />
  </div>

  <ul id="command-list" role="listbox" aria-label="Commands">
    <li role="option" id="cmd-1">
      <kbd>Ctrl+N</kbd> New file
    </li>
    <li role="option" id="cmd-2">
      <kbd>Ctrl+O</kbd> Open file
    </li>
    <li role="option" id="cmd-3" aria-selected="true">
      <kbd>Ctrl+S</kbd> Save
    </li>
  </ul>

  <div class="command-footer">
    <span>↑↓ Navigate</span>
    <span>↵ Select</span>
    <span>Esc Close</span>
  </div>
</div>
```

### Keyboard Interaction

| Key | Action |
|-----|--------|
| `Ctrl + K` or `/` | Open palette (global shortcut) |
| `Arrow Down/Up` | Navigate options |
| `Enter` | Execute selected command |
| `Escape` | Close palette |
| Type | Filter commands |
| `Backspace` on empty | Go back to previous category (if nested) |
