# Advanced Form Patterns Reference

## Dynamic Forms

### Add/Remove Fields

```html
<fieldset aria-label="Team members">
  <legend>Team Members</legend>

  <div class="field-group" data-index="0">
    <label for="member-0">Member name</label>
    <input id="member-0" type="text" name="members[]" required />
    <button type="button" aria-label="Remove member" class="remove-field">
      Remove
    </button>
  </div>

  <button type="button" id="add-member" aria-describedby="member-limit">
    + Add member
  </button>
  <p id="member-limit" class="helper-text">Maximum 10 members</p>
</fieldset>
```

**Accessibility requirements:**
- Announce additions: `aria-live="polite"` region that says "Member field added"
- On remove: move focus to the previous field or the add button
- Disable "Add" when maximum reached; show remaining count
- Each field must have a unique `id` and associated `label`

### Conditional Field Visibility

```html
<div class="form-group">
  <label for="contact-method">Preferred contact method</label>
  <select id="contact-method" aria-controls="contact-details">
    <option value="">Select...</option>
    <option value="email">Email</option>
    <option value="phone">Phone</option>
    <option value="mail">Postal mail</option>
  </select>
</div>

<!-- Conditionally shown sections -->
<div id="contact-details">
  <div id="email-fields" hidden>
    <label for="email">Email address</label>
    <input id="email" type="email" />
  </div>

  <div id="phone-fields" hidden>
    <label for="phone">Phone number</label>
    <input id="phone" type="tel" />
  </div>

  <div id="mail-fields" hidden>
    <label for="address">Street address</label>
    <input id="address" type="text" />
  </div>
</div>
```

**Implementation rules:**
- Use `hidden` attribute (not `display:none` from CSS) for fields that shouldn't submit
- Hidden fields must not be `required` — toggle required dynamically
- Animate visibility with height/opacity for smooth transitions
- Announce changes: "Email fields now visible"

### Dependent Field Cascades

```javascript
// Country → State → City cascade
countrySelect.addEventListener('change', async (e) => {
  const country = e.target.value;

  // Reset dependent fields
  stateSelect.innerHTML = '<option value="">Loading...</option>';
  stateSelect.disabled = true;
  citySelect.innerHTML = '<option value="">Select state first</option>';
  citySelect.disabled = true;

  const states = await fetchStates(country);
  stateSelect.innerHTML = '<option value="">Select state</option>' +
    states.map(s => `<option value="${s.id}">${s.name}</option>`).join('');
  stateSelect.disabled = false;

  // Announce to screen readers
  liveRegion.textContent = `${states.length} states loaded for ${country}`;
});
```

---

## Wizard / Stepper Flows

### Step Indicator Pattern

```html
<nav aria-label="Checkout progress">
  <ol class="stepper">
    <li class="stepper__step stepper__step--completed">
      <a href="#step-1" aria-label="Step 1: Shipping, completed">
        <span class="stepper__number">1</span>
        <span class="stepper__label">Shipping</span>
      </a>
    </li>
    <li class="stepper__step stepper__step--active" aria-current="step">
      <span class="stepper__number">2</span>
      <span class="stepper__label">Payment</span>
    </li>
    <li class="stepper__step stepper__step--upcoming">
      <span class="stepper__number">3</span>
      <span class="stepper__label">Review</span>
    </li>
  </ol>
</nav>
```

### Non-Linear Wizard

Users can navigate to any completed step or the current step, but not future steps.

```css
.stepper__step--completed a {
  cursor: pointer;
  color: var(--color-primary);
}

.stepper__step--upcoming {
  opacity: 0.5;
  pointer-events: none;
}
```

### State Persistence Between Steps

```javascript
// Persist form state to sessionStorage on every change
function persistStepData(stepId, data) {
  const wizardState = JSON.parse(sessionStorage.getItem('wizard') || '{}');
  wizardState[stepId] = { ...wizardState[stepId], ...data };
  wizardState._lastStep = stepId;
  sessionStorage.setItem('wizard', JSON.stringify(wizardState));
}

// Restore on page load or step navigation
function restoreStepData(stepId) {
  const wizardState = JSON.parse(sessionStorage.getItem('wizard') || '{}');
  return wizardState[stepId] || {};
}
```

**Wizard UX Rules:**
- Validate the current step before allowing progression
- Allow navigating back to completed steps without data loss
- Show a review/summary step before final submission
- Persist data across browser refresh (sessionStorage minimum)
- Show clear step count ("Step 2 of 4")

---

## Auto-Save Patterns

### Debounced Auto-Save

```javascript
function createAutoSave(saveFunction, delay = 1500) {
  let timeoutId;
  let isSaving = false;

  return {
    trigger(data) {
      clearTimeout(timeoutId);
      showSaveIndicator('unsaved');

      timeoutId = setTimeout(async () => {
        if (isSaving) return;
        isSaving = true;
        showSaveIndicator('saving');

        try {
          await saveFunction(data);
          showSaveIndicator('saved');
        } catch (error) {
          showSaveIndicator('error');
        } finally {
          isSaving = false;
        }
      }, delay);
    }
  };
}
```

### Save Indicator States

```html
<div class="save-indicator" aria-live="polite" aria-atomic="true">
  <!-- States cycle through: -->
  <span class="save-indicator--unsaved">Unsaved changes</span>
  <span class="save-indicator--saving">Saving...</span>
  <span class="save-indicator--saved">All changes saved</span>
  <span class="save-indicator--error">
    Save failed. <button>Retry</button>
  </span>
</div>
```

### Conflict Resolution

When two users edit the same resource:

1. **Last write wins** — Simple but can lose data. Acceptable for non-critical content.
2. **Optimistic locking** — Check version before saving. Show conflict dialog if version mismatch.
3. **Real-time merge** — Use operational transforms or CRDTs for collaborative editing.

```javascript
async function saveWithConflictDetection(data, expectedVersion) {
  try {
    const response = await api.save(data, { ifVersion: expectedVersion });
    return { success: true, newVersion: response.version };
  } catch (error) {
    if (error.status === 409) {
      // Version conflict
      const serverData = await api.get(data.id);
      showConflictDialog(data, serverData);
      return { success: false, conflict: true };
    }
    throw error;
  }
}
```

---

## Accessible Drag-and-Drop

### Keyboard Alternative

Every drag-and-drop interaction must have a keyboard alternative.

```html
<ul role="listbox" aria-label="Task list (reorderable)">
  <li role="option" aria-selected="false" tabindex="0" aria-roledescription="sortable item">
    <span class="drag-handle" aria-hidden="true">⠿</span>
    <span>Task name</span>
    <div class="reorder-controls">
      <button aria-label="Move up" class="move-up">↑</button>
      <button aria-label="Move down" class="move-down">↓</button>
    </div>
  </li>
</ul>

<!-- Live region for announcements -->
<div aria-live="assertive" class="sr-only" id="drag-announce"></div>
```

### Screen Reader Announcements During Drag

```javascript
function announceGrab(item, position, total) {
  announce(`Grabbed ${item.textContent}. Current position: ${position} of ${total}. Use arrow keys to move, Space to drop, Escape to cancel.`);
}

function announceMove(item, newPosition, total) {
  announce(`${item.textContent} moved to position ${newPosition} of ${total}.`);
}

function announceDrop(item, finalPosition) {
  announce(`${item.textContent} dropped at position ${finalPosition}.`);
}

function announceCancel(item, originalPosition) {
  announce(`Reorder cancelled. ${item.textContent} returned to position ${originalPosition}.`);
}
```

### Drop Zone Indicators

```css
.drop-zone {
  border: 2px dashed var(--color-border-default);
  border-radius: var(--radius-md);
  padding: var(--space-8);
  text-align: center;
  transition: border-color var(--duration-fast), background-color var(--duration-fast);
}

.drop-zone--active {
  border-color: var(--color-primary);
  background-color: var(--color-bg-accent-subtle);
}

.drop-zone--invalid {
  border-color: var(--color-border-error);
  background-color: var(--color-bg-error-subtle);
}
```

---

## File Upload Patterns

### Drag-and-Drop Upload Area

```html
<div class="upload-area"
     role="button"
     tabindex="0"
     aria-label="Upload files. Drag and drop or click to browse. Accepted: JPG, PNG, PDF. Max 10MB.">
  <input type="file" id="file-input" multiple accept=".jpg,.png,.pdf" class="sr-only" />

  <div class="upload-area__content">
    <svg class="upload-icon" aria-hidden="true"><!-- upload icon --></svg>
    <p><strong>Drag files here</strong> or <button type="button" class="link-button">browse</button></p>
    <p class="helper-text">JPG, PNG, PDF up to 10MB</p>
  </div>
</div>

<!-- Upload progress list -->
<ul class="upload-list" aria-label="Uploading files" aria-live="polite">
  <li class="upload-item">
    <span class="upload-item__name">document.pdf</span>
    <progress value="65" max="100" aria-label="document.pdf upload progress">65%</progress>
    <button aria-label="Cancel upload for document.pdf">Cancel</button>
  </li>
</ul>
```

### Upload States

| State | Visual | ARIA |
|-------|--------|------|
| Idle | Dashed border, upload icon | `aria-label` describes accepted formats |
| Drag over | Highlighted border, "Drop here" text | — |
| Uploading | Progress bar per file | `<progress>` with label per file |
| Complete | Checkmark, file preview | Live region: "file.pdf uploaded successfully" |
| Error | Error message, retry button | `role="alert"` for error message |

---

## Inline Editing

### Click-to-Edit Pattern

```html
<!-- Display mode -->
<div class="editable" role="button" tabindex="0"
     aria-label="Project name: My Project. Click to edit.">
  <span class="editable__value">My Project</span>
  <svg class="editable__icon" aria-hidden="true"><!-- pencil icon --></svg>
</div>

<!-- Edit mode (toggled) -->
<div class="editable editable--editing">
  <input type="text" value="My Project" aria-label="Project name"
         class="editable__input" />
  <button aria-label="Save" class="editable__save">✓</button>
  <button aria-label="Cancel" class="editable__cancel">✕</button>
</div>
```

**Keyboard interaction:**
- `Enter` or `Click`: Enter edit mode, focus input, select all text
- `Enter` (in input): Save and exit edit mode
- `Escape`: Cancel edit, restore original value, return focus to display element
- `Tab`: Save and move focus to next element

---

## Search and Filter Forms

### Filter Chip Pattern

```html
<div class="active-filters" aria-label="Active filters">
  <span class="filter-chip">
    Status: Active
    <button aria-label="Remove filter: Status Active" class="filter-chip__remove">×</button>
  </span>
  <span class="filter-chip">
    Priority: High
    <button aria-label="Remove filter: Priority High" class="filter-chip__remove">×</button>
  </span>
  <button class="clear-all-filters">Clear all filters</button>
</div>

<!-- Results count (announced on change) -->
<div aria-live="polite" aria-atomic="true" class="results-count">
  Showing 24 of 156 results
</div>
```

### Search-as-you-type

```javascript
const searchInput = document.getElementById('search');
let debounceTimer;

searchInput.addEventListener('input', (e) => {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(async () => {
    const query = e.target.value.trim();
    if (query.length < 2) return; // Minimum query length

    const results = await search(query);
    renderResults(results);

    // Announce result count
    liveRegion.textContent = `${results.length} results found for "${query}"`;
  }, 300); // 300ms debounce
});
```
