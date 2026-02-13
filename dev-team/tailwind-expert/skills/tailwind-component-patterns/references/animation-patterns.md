# Animation Patterns in Tailwind CSS

## Transition Utilities

To add smooth transitions to interactive elements, apply one of the transition property utilities along with duration and timing controls.

### Transition Properties

To transition all animatable properties at once, use `transition-all`. To limit transitions to specific property groups, choose a scoped utility:

| Utility                | Properties Transitioned                            |
|------------------------|----------------------------------------------------|
| `transition-all`       | All properties                                     |
| `transition-colors`    | Color, background-color, border-color, fill, stroke, text-decoration-color |
| `transition-opacity`   | Opacity                                            |
| `transition-transform` | Transform (translate, scale, rotate)               |
| `transition-shadow`    | Box-shadow                                         |
| `transition-none`      | Disables transitions entirely                      |

To apply `transition-colors` to a button, pair it with a hover state change so the color shift is animated rather than instant:

```html
<button class="bg-blue-600 hover:bg-blue-700 text-white transition-colors">
  Save
</button>
```

### Duration

To control how long a transition takes, append a duration utility:

- `duration-75` -- 75ms, nearly instant
- `duration-100` -- 100ms
- `duration-150` -- 150ms (Tailwind default)
- `duration-200` -- 200ms, a natural feel for color changes
- `duration-300` -- 300ms, suitable for transforms and layout shifts
- `duration-500` -- 500ms, deliberate and visible
- `duration-700` -- 700ms
- `duration-1000` -- 1000ms, use sparingly for dramatic effects

For most interactive elements, 150ms to 300ms produces the most natural feel. To apply a 200ms duration to a color transition:

```html
<a class="text-gray-600 hover:text-blue-600 transition-colors duration-200">Link</a>
```

### Timing Functions

To control the acceleration curve of a transition, apply an easing utility:

- `ease-linear` -- constant speed, useful for progress bars and spinners
- `ease-in` -- starts slow, accelerates, suitable for elements leaving the viewport
- `ease-out` -- starts fast, decelerates, best for elements entering the viewport
- `ease-in-out` -- slow start and end, smooth for hover state changes

The common pattern for interactive elements combines all three concerns:

```html
<div class="transition-colors duration-200 ease-in-out hover:bg-gray-100">
  Hover me
</div>
```

### Delay

To postpone the start of a transition, add a delay utility:

- `delay-0` -- no delay
- `delay-75` -- 75ms
- `delay-100` -- 100ms
- `delay-150` -- 150ms
- `delay-200` -- 200ms
- `delay-300` -- 300ms
- `delay-500` -- 500ms
- `delay-700` -- 700ms
- `delay-1000` -- 1s

To create a staggered entrance effect across a list of items, apply incrementing delays:

```html
<ul>
  <li class="opacity-0 animate-[fadeIn_0.3s_ease-out_forwards] delay-0">Item 1</li>
  <li class="opacity-0 animate-[fadeIn_0.3s_ease-out_forwards] delay-100">Item 2</li>
  <li class="opacity-0 animate-[fadeIn_0.3s_ease-out_forwards] delay-200">Item 3</li>
  <li class="opacity-0 animate-[fadeIn_0.3s_ease-out_forwards] delay-300">Item 4</li>
</ul>
```

## Built-in Animations

Tailwind provides four built-in animation utilities that handle common patterns without any custom configuration.

### `animate-spin`

To indicate a loading state, apply `animate-spin` to a circular SVG icon. The element rotates 360 degrees in a continuous linear loop:

```html
<svg class="animate-spin h-5 w-5 text-blue-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
  <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
  <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
</svg>
```

### `animate-ping`

To draw attention to a notification badge or status indicator, apply `animate-ping`. The element scales up and fades out in a pulsating ring effect:

```html
<span class="relative flex h-3 w-3">
  <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-red-400 opacity-75"></span>
  <span class="relative inline-flex rounded-full h-3 w-3 bg-red-500"></span>
</span>
```

### `animate-pulse`

To build skeleton loading screens, apply `animate-pulse` to placeholder elements. The opacity cycles between 100% and 50%, creating a subtle breathing effect:

```html
<div class="animate-pulse space-y-4">
  <div class="h-4 bg-gray-200 rounded w-3/4"></div>
  <div class="h-4 bg-gray-200 rounded w-1/2"></div>
  <div class="h-10 bg-gray-200 rounded"></div>
</div>
```

### `animate-bounce`

To highlight a call-to-action element such as a scroll-down arrow, apply `animate-bounce`. The element moves vertically with an eased bounce:

```html
<div class="animate-bounce w-10 h-10 flex items-center justify-center">
  <svg class="w-6 h-6 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 14l-7 7m0 0l-7-7m7 7V3"></path>
  </svg>
</div>
```

## Custom Keyframe Animations

### Tailwind v3: Configuration in tailwind.config.js

To define custom animations in Tailwind v3, extend the theme in `tailwind.config.js` by adding keyframes and referencing them in the animation key:

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { opacity: '0', transform: 'translateY(10px)' },
          '100%': { opacity: '1', transform: 'translateY(0)' },
        },
        scaleIn: {
          '0%': { opacity: '0', transform: 'scale(0.95)' },
          '100%': { opacity: '1', transform: 'scale(1)' },
        },
      },
      animation: {
        fadeIn: 'fadeIn 0.3s ease-out forwards',
        slideUp: 'slideUp 0.3s ease-out forwards',
        scaleIn: 'scaleIn 0.2s ease-out forwards',
      },
    },
  },
};
```

To use these animations in markup, apply the generated utility classes:

```html
<div class="animate-fadeIn">Fades in on mount</div>
<div class="animate-slideUp">Slides up on mount</div>
<div class="animate-scaleIn">Scales in on mount</div>
```

### Tailwind v4: CSS-Native Configuration

To define custom animations in Tailwind v4, declare keyframes and animation theme values directly in CSS using `@theme`:

```css
@theme {
  --animate-fade-in: fade-in 0.3s ease-out forwards;
  --animate-slide-up: slide-up 0.3s ease-out forwards;
  --animate-scale-in: scale-in 0.2s ease-out forwards;
}

@keyframes fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes slide-up {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes scale-in {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}
```

To reference these in markup, use the automatically generated utility classes:

```html
<div class="animate-fade-in">Fades in</div>
<div class="animate-slide-up">Slides up</div>
<div class="animate-scale-in">Scales in</div>
```

## Enter/Leave Transitions

### Pattern for Show/Hide Elements

To animate elements like dropdown menus and modals when they appear and disappear, manage two states -- entering and leaving -- each with distinct starting and ending classes.

To implement a basic enter/leave pattern with data attributes, toggle a `data-state` attribute between `open` and `closed` via JavaScript and style accordingly:

```html
<div
  data-state="closed"
  class="transition-all duration-200 ease-out data-[state=open]:opacity-100 data-[state=open]:scale-100 data-[state=closed]:opacity-0 data-[state=closed]:scale-95 data-[state=closed]:pointer-events-none"
>
  Dropdown content
</div>
```

To toggle visibility, update the `data-state` attribute in JavaScript:

```js
function toggleDropdown(el) {
  const isOpen = el.dataset.state === 'open';
  el.dataset.state = isOpen ? 'closed' : 'open';
}
```

### Integration with Headless UI

To use Headless UI's built-in transition component with Tailwind classes, pass enter/leave class sets:

```jsx
<Transition
  show={isOpen}
  enter="transition ease-out duration-200"
  enterFrom="opacity-0 scale-95"
  enterTo="opacity-100 scale-100"
  leave="transition ease-in duration-150"
  leaveFrom="opacity-100 scale-100"
  leaveTo="opacity-0 scale-95"
>
  <div class="absolute mt-2 w-48 bg-white rounded-lg shadow-lg">
    {/* dropdown items */}
  </div>
</Transition>
```

### Integration with Radix UI

To animate Radix primitives, target the `data-[state=open]` and `data-[state=closed]` attributes that Radix automatically applies:

```html
<div class="data-[state=open]:animate-fade-in data-[state=closed]:animate-fade-out">
  Radix-managed content
</div>
```

### Native `<dialog>` Element

To animate the native `<dialog>` element, use the `open` attribute for styling and pair `showModal()` / `close()` with CSS transitions:

```css
dialog {
  opacity: 0;
  transform: scale(0.95);
  transition: opacity 0.2s ease-out, transform 0.2s ease-out, overlay 0.2s ease-out allow-discrete, display 0.2s ease-out allow-discrete;
}

dialog[open] {
  opacity: 1;
  transform: scale(1);
}

dialog::backdrop {
  background: rgb(0 0 0 / 0);
  transition: background 0.2s ease-out, overlay 0.2s ease-out allow-discrete, display 0.2s ease-out allow-discrete;
}

dialog[open]::backdrop {
  background: rgb(0 0 0 / 0.5);
}

@starting-style {
  dialog[open] {
    opacity: 0;
    transform: scale(0.95);
  }
  dialog[open]::backdrop {
    background: rgb(0 0 0 / 0);
  }
}
```

To trigger the dialog, call `dialogRef.showModal()` in JavaScript. The browser handles focus trapping and backdrop display automatically.

## Reduced Motion

To respect user accessibility preferences, always pair animations with reduced motion variants. Users who enable "Reduce motion" in their operating system settings trigger the `prefers-reduced-motion: reduce` media query.

### Disabling Animations

To disable transitions for users who prefer reduced motion, apply `motion-reduce:transition-none`:

```html
<button class="transition-colors duration-200 motion-reduce:transition-none hover:bg-blue-700">
  Click me
</button>
```

To disable keyframe animations, apply `motion-reduce:animate-none`:

```html
<div class="animate-bounce motion-reduce:animate-none">
  Scroll down
</div>
```

### Opting In with motion-safe

To apply animations only when the user has not requested reduced motion, use `motion-safe:` instead:

```html
<div class="motion-safe:animate-bounce">
  Only bounces if motion is allowed
</div>

<button class="motion-safe:transition-colors motion-safe:duration-200 hover:bg-blue-700">
  Only transitions if motion is allowed
</button>
```

The `motion-safe:` approach is often preferable because it treats no-animation as the default and opts in only when safe. This ensures that forgetting to add a motion-reduce variant does not result in unwanted animation for sensitive users.

### Best Practices for Reduced Motion

To handle reduced motion comprehensively:

1. Apply `motion-reduce:transition-none` or `motion-reduce:animate-none` to every animated element.
2. Alternatively, adopt a motion-safe-first approach where animations are only applied via `motion-safe:` prefixes.
3. For essential animations that convey information (such as a loading spinner), keep them but reduce intensity -- for example, replace `animate-spin` with a static loading icon or a very slow, subtle pulse.
4. Never rely solely on animation to communicate state changes. Pair visual motion with text updates, icon changes, or ARIA live regions.

## Scroll-Driven Animations

### Using Arbitrary Properties for Scroll-Timeline

To create scroll-driven animations using modern CSS features, apply arbitrary properties for scroll-timeline binding. Note that browser support is still evolving, so provide fallbacks:

```html
<div class="[animation-timeline:scroll()] [animation-name:progress] [animation-duration:auto] [animation-timing-function:linear]">
  <div class="h-1 bg-blue-600 origin-left"></div>
</div>
```

Define the corresponding keyframes in CSS:

```css
@keyframes progress {
  from {
    transform: scaleX(0);
  }
  to {
    transform: scaleX(1);
  }
}
```

This creates a scroll progress bar that fills as the user scrolls down the page.

### Intersection Observer Pattern with Tailwind

To trigger animations when elements scroll into view, use the Intersection Observer API to toggle Tailwind classes via JavaScript:

```js
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add('opacity-100', 'translate-y-0');
        entry.target.classList.remove('opacity-0', 'translate-y-4');
      }
    });
  },
  { threshold: 0.1 }
);

document.querySelectorAll('[data-animate-on-scroll]').forEach((el) => {
  observer.observe(el);
});
```

To prepare elements for this pattern, set initial hidden state classes and the transition configuration in the markup:

```html
<div
  data-animate-on-scroll
  class="opacity-0 translate-y-4 transition-all duration-500 ease-out"
>
  This content animates into view on scroll.
</div>
```

To create a staggered scroll animation across multiple elements, apply incremental `delay-` utilities:

```html
<div data-animate-on-scroll class="opacity-0 translate-y-4 transition-all duration-500 ease-out delay-0">Card 1</div>
<div data-animate-on-scroll class="opacity-0 translate-y-4 transition-all duration-500 ease-out delay-100">Card 2</div>
<div data-animate-on-scroll class="opacity-0 translate-y-4 transition-all duration-500 ease-out delay-200">Card 3</div>
```

To handle elements that should animate every time they enter the viewport (not just once), add logic to remove classes when the element leaves:

```js
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add('opacity-100', 'translate-y-0');
        entry.target.classList.remove('opacity-0', 'translate-y-4');
      } else {
        entry.target.classList.remove('opacity-100', 'translate-y-0');
        entry.target.classList.add('opacity-0', 'translate-y-4');
      }
    });
  },
  { threshold: 0.1 }
);
```

To ensure these scroll-driven patterns degrade gracefully for users who prefer reduced motion, wrap the observer initialization in a media query check:

```js
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (!prefersReducedMotion) {
  document.querySelectorAll('[data-animate-on-scroll]').forEach((el) => {
    observer.observe(el);
  });
} else {
  // Make all elements visible immediately without animation
  document.querySelectorAll('[data-animate-on-scroll]').forEach((el) => {
    el.classList.remove('opacity-0', 'translate-y-4');
  });
}
```
