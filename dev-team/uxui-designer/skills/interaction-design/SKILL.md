---
name: Interaction Design
description: This skill should be used when the user asks about "micro-interaction", "animation design", "loading pattern", "form UX", "error handling UX", "navigation pattern", "state transition", "skeleton screen", or "user flow". It covers animation specifications, state machine design, loading and error patterns, form interaction design, and navigation architecture.
---

### Animation Specification

Every animation must serve one of four purposes:

1. **Feedback** — Confirm an action occurred (button press, toggle switch).
2. **Orientation** — Show spatial relationships (slide-in panel, page transition).
3. **Focus** — Direct attention to important content (notification pulse, highlight).
4. **Continuity** — Maintain context during state changes (morphing card to detail view).

#### Timing Guidelines

| Purpose | Duration | Easing |
|---------|----------|--------|
| Micro-feedback (button, toggle) | 100-150ms | `ease-out` |
| Small transitions (dropdown, tooltip) | 150-250ms | `ease-out` |
| Medium transitions (modal, drawer) | 200-350ms | `ease-in-out` or `cubic-bezier(0.4, 0, 0.2, 1)` |
| Large transitions (page, full-screen) | 300-500ms | `cubic-bezier(0.4, 0, 0.2, 1)` |
| Exit animations | 20-30% faster than enter | `ease-in` |

```css
/* Standard easing tokens */
--ease-default: cubic-bezier(0.4, 0, 0.2, 1);   /* Material standard */
--ease-in: cubic-bezier(0.4, 0, 1, 1);
--ease-out: cubic-bezier(0, 0, 0.2, 1);
--ease-spring: cubic-bezier(0.175, 0.885, 0.32, 1.275); /* Slight overshoot */

/* Duration tokens */
--duration-fast: 150ms;
--duration-normal: 250ms;
--duration-slow: 350ms;

/* Respect user preferences */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

### State Transition Design

Design every interactive element as a state machine with defined transitions.

#### Button States

```
[Default] → hover → [Hover]
[Default] → focus-visible → [Focus]
[Default] → mousedown → [Active/Pressed]
[Default] → disabled → [Disabled]
[Default] → click → [Loading] → success → [Default]
                               → error → [Error] → timeout → [Default]
```

For each state, specify:
- Visual changes (background, border, shadow, scale, opacity)
- Transition properties and duration
- Cursor style
- ARIA attributes (aria-disabled, aria-busy)

#### Async Action Pattern

```
[Idle]
  ↓ trigger action
[Loading]
  → Show spinner/progress indicator
  → Disable trigger to prevent duplicate submissions
  → Maintain layout stability (fixed dimensions)
  ↓ success
[Success]
  → Brief success indicator (checkmark, green flash)
  → Auto-dismiss after 1.5-2s
  → Return to [Idle]
  ↓ error
[Error]
  → Inline error message near trigger
  → Clear error message with retry affordance
  → Do not auto-dismiss errors (user must acknowledge)
```

### Loading Patterns

Choose the appropriate loading pattern based on expected wait time:

| Wait Time | Pattern | Implementation |
|-----------|---------|----------------|
| <300ms | No indicator | Delay showing any loading UI |
| 300ms-1s | Skeleton screen | Layout-preserving placeholder shapes |
| 1-3s | Skeleton + progress | Skeleton with subtle shimmer animation |
| 3-10s | Determinate progress | Progress bar with percentage or steps |
| >10s | Progress + status text | "Processing step 2 of 5..." |
| Unknown | Indeterminate + explanation | "This may take a few minutes..." |

#### Skeleton Screen Pattern

```css
.skeleton {
  background: linear-gradient(
    90deg,
    var(--color-gray-100) 25%,
    var(--color-gray-200) 50%,
    var(--color-gray-100) 75%
  );
  background-size: 200% 100%;
  animation: skeleton-shimmer 1.5s ease-in-out infinite;
  border-radius: var(--radius-sm);
}

@keyframes skeleton-shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

@media (prefers-reduced-motion: reduce) {
  .skeleton {
    animation: none;
    background: var(--color-gray-200);
  }
}
```

Match skeleton shapes to actual content layout. Avoid generic loading spinners when the content structure is predictable.

### Form Interaction Design

#### Validation Timing

| Validation Trigger | When to Use |
|-------------------|-------------|
| On submit | Default for most forms. Validate all fields at once. |
| On blur (field exit) | Complex fields where early feedback saves time (email, URL, password strength). |
| On change (real-time) | Only for character counters, password strength meters, or search-as-you-type. |
| Hybrid | Validate on submit first; after first error, switch to on-blur for corrected fields. |

#### Error Message Placement

- Place error messages directly below the input field.
- Use `aria-describedby` to associate error with input.
- Use `role="alert"` or `aria-live="polite"` for dynamically appearing errors.
- Color-code errors (red border + icon) but never rely on color alone — include text.

```html
<div class="form-field form-field--error">
  <label for="email">Email address</label>
  <input id="email" type="email" aria-describedby="email-error" aria-invalid="true" />
  <p id="email-error" class="form-field__error" role="alert">
    Enter a valid email address (e.g., name@example.com)
  </p>
</div>
```

#### Multi-Step Form Patterns

- Show a step indicator with current position and total steps.
- Allow navigation to completed steps (never lock users out of reviewing).
- Persist data between steps (no data loss on back-navigation).
- Validate each step before allowing progression.
- Show a review summary before final submission.

### Navigation Patterns

#### Primary Navigation Selection

| Pattern | Use When | Avoid When |
|---------|----------|------------|
| Top bar | 3-7 top-level items, desktop-first | >7 items, deep hierarchy |
| Side nav | >7 items, deep hierarchy, dashboard apps | Content-focused sites, mobile-first |
| Bottom tab bar | Mobile apps, 3-5 top-level destinations | >5 items, desktop apps |
| Breadcrumbs | Deep hierarchy, user needs orientation | Flat site structure |
| Hamburger menu | Mobile viewport, secondary navigation | Desktop primary nav (hides discoverability) |

#### Navigation Transitions

Communicate spatial relationships through transition direction:
- **Forward navigation** (deeper): Slide content left, new content enters from right.
- **Back navigation** (shallower): Slide content right, previous content enters from left.
- **Lateral navigation** (same level): Crossfade or no transition.
- **Modal/overlay**: Scale up from trigger point or slide up from bottom.

### Empty States

Design empty states that guide users toward action:

1. **Illustration or icon** — Visual that matches the feature's purpose.
2. **Headline** — What the user sees when content exists (e.g., "No projects yet").
3. **Description** — Brief explanation of the feature's value.
4. **Primary action** — Clear CTA to create the first item (e.g., "Create your first project").

```
┌─────────────────────────────┐
│       [Illustration]        │
│                             │
│     No projects yet         │
│                             │
│  Create a project to start  │
│  organizing your work.      │
│                             │
│  [+ Create Project]         │
└─────────────────────────────┘
```

Never show a blank screen. Every empty state is an onboarding opportunity.

## References

- [animation-patterns.md](references/animation-patterns.md) - Complete animation library with spring physics, orchestrated sequences, scroll-triggered animations, view transitions, and motion design systems.
- [form-patterns.md](references/form-patterns.md) - Advanced form patterns including dynamic forms, conditional fields, wizard flows, auto-save, drag-and-drop reordering, and complex validation scenarios.
