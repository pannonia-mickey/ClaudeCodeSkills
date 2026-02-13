# Animation Patterns Reference

## Spring Physics Animations

Spring animations produce natural, organic motion by simulating physical spring behavior. Unlike cubic-bezier curves, springs have no fixed duration — they settle when the system reaches equilibrium.

### Spring Parameters

| Parameter | Description | Typical Range |
|-----------|-------------|---------------|
| **stiffness** | How taut the spring is (higher = snappier) | 100-1000 |
| **damping** | Friction that slows oscillation (higher = less bounce) | 10-100 |
| **mass** | Weight of the object (higher = slower, heavier feel) | 0.5-3 |

### CSS Spring (Using `linear()` Approximation)

```css
/* Bouncy spring: stiffness=300, damping=15 */
--ease-spring-bouncy: linear(
  0, 0.006, 0.025, 0.056, 0.099, 0.153, 0.218, 0.293,
  0.377, 0.468, 0.565, 0.666, 0.77, 0.872, 0.971,
  1.062, 1.143, 1.211, 1.264, 1.3, 1.32, 1.324,
  1.315, 1.294, 1.264, 1.229, 1.192, 1.155, 1.12,
  1.088, 1.061, 1.04, 1.023, 1.012, 1.004, 1
);

/* Smooth spring: stiffness=200, damping=25 */
--ease-spring-smooth: linear(
  0, 0.009, 0.035, 0.078, 0.135, 0.205, 0.286, 0.376,
  0.471, 0.569, 0.666, 0.759, 0.845, 0.921, 0.984,
  1.033, 1.068, 1.089, 1.098, 1.098, 1.091, 1.078,
  1.063, 1.046, 1.03, 1.016, 1.005, 0.998, 0.994,
  0.993, 0.995, 0.998, 1
);

.modal-enter {
  animation: modal-in 600ms var(--ease-spring-bouncy) forwards;
}

@keyframes modal-in {
  from { transform: scale(0.9); opacity: 0; }
  to { transform: scale(1); opacity: 1; }
}
```

### JavaScript Spring (Web Animations API)

```javascript
// Simple spring simulation
function springAnimation(element, target, { stiffness = 300, damping = 20, mass = 1 }) {
  let position = 0;
  let velocity = 0;
  const threshold = 0.01;

  function step() {
    const force = -stiffness * (position - target);
    const dampingForce = -damping * velocity;
    const acceleration = (force + dampingForce) / mass;

    velocity += acceleration * (1 / 60);
    position += velocity * (1 / 60);

    element.style.transform = `translateY(${position}px)`;

    if (Math.abs(velocity) > threshold || Math.abs(position - target) > threshold) {
      requestAnimationFrame(step);
    }
  }

  requestAnimationFrame(step);
}
```

---

## Orchestrated Animation Sequences

### Staggered Entrance

Items enter sequentially with a fixed delay between each.

```css
.list-item {
  opacity: 0;
  transform: translateY(20px);
  animation: fade-up 400ms var(--ease-out) forwards;
}

/* Stagger each item by 50ms */
.list-item:nth-child(1) { animation-delay: 0ms; }
.list-item:nth-child(2) { animation-delay: 50ms; }
.list-item:nth-child(3) { animation-delay: 100ms; }
.list-item:nth-child(4) { animation-delay: 150ms; }

/* Or use custom properties for dynamic stagger */
.list-item {
  animation-delay: calc(var(--index) * 50ms);
}

@keyframes fade-up {
  to { opacity: 1; transform: translateY(0); }
}
```

```javascript
// Set stagger index via JS
document.querySelectorAll('.list-item').forEach((item, index) => {
  item.style.setProperty('--index', index);
});
```

### Choreographed Multi-Element Transitions

```css
/* Page transition: header slides down, content fades up, sidebar slides in */
.page-enter .header {
  animation: slide-down 300ms var(--ease-out) forwards;
}

.page-enter .main-content {
  animation: fade-up 400ms var(--ease-out) 150ms forwards; /* 150ms delay */
}

.page-enter .sidebar {
  animation: slide-right 350ms var(--ease-out) 200ms forwards; /* 200ms delay */
}
```

### Orchestration Principles

1. **Lead with the most important element** — Primary content enters first
2. **50-100ms between elements** — Enough to perceive sequence, not enough to feel slow
3. **Total sequence under 800ms** — Beyond this, users perceive the animation as slow
4. **Exit faster than enter** — Exit animations should be 20-30% shorter
5. **Exits can be simultaneous** — Unlike entrances, departing elements can exit together

---

## Scroll-Triggered Animations

### Intersection Observer Pattern

```javascript
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add('is-visible');
        observer.unobserve(entry.target); // Animate only once
      }
    });
  },
  { threshold: 0.2, rootMargin: '0px 0px -50px 0px' }
);

document.querySelectorAll('[data-animate]').forEach((el) => observer.observe(el));
```

```css
[data-animate] {
  opacity: 0;
  transform: translateY(30px);
  transition: opacity 600ms var(--ease-out), transform 600ms var(--ease-out);
}

[data-animate].is-visible {
  opacity: 1;
  transform: translateY(0);
}
```

### CSS Scroll-Driven Animations API

```css
/* Progress bar that fills as user scrolls the page */
.scroll-progress {
  position: fixed;
  top: 0;
  left: 0;
  height: 3px;
  background: var(--color-primary);
  transform-origin: left;
  animation: scroll-fill linear both;
  animation-timeline: scroll(root block);
}

@keyframes scroll-fill {
  from { transform: scaleX(0); }
  to { transform: scaleX(1); }
}

/* Element fades in as it enters viewport */
.fade-on-scroll {
  animation: fade-in linear both;
  animation-timeline: view();
  animation-range: entry 0% entry 100%;
}

@keyframes fade-in {
  from { opacity: 0; transform: translateY(50px); }
  to { opacity: 1; transform: translateY(0); }
}
```

### Parallax with Scroll-Driven Animations

```css
.parallax-bg {
  animation: parallax linear both;
  animation-timeline: scroll(root);
}

@keyframes parallax {
  from { transform: translateY(0); }
  to { transform: translateY(-30%); }
}
```

---

## View Transitions API

### SPA Route Transitions

```javascript
// Wrap navigation in startViewTransition
document.addEventListener('click', async (e) => {
  const link = e.target.closest('a[data-transition]');
  if (!link) return;

  e.preventDefault();

  if (!document.startViewTransition) {
    navigateTo(link.href);
    return;
  }

  const transition = document.startViewTransition(() => {
    navigateTo(link.href);
  });

  await transition.finished;
});
```

```css
/* Default crossfade */
::view-transition-old(root) {
  animation: fade-out 200ms ease-in;
}

::view-transition-new(root) {
  animation: fade-in 300ms ease-out;
}

/* Named transition for a shared element */
.product-image {
  view-transition-name: product-hero;
}

::view-transition-old(product-hero),
::view-transition-new(product-hero) {
  animation-duration: 400ms;
  animation-timing-function: var(--ease-spring-smooth);
}
```

### Morph Transitions (Shared Element)

```css
/* Card thumbnail morphs into full image on detail page */
.card .thumbnail { view-transition-name: hero-image; }
.detail .hero { view-transition-name: hero-image; }

/* Card title morphs into page heading */
.card .title { view-transition-name: page-title; }
.detail h1 { view-transition-name: page-title; }
```

---

## Motion Design System

### Motion Token Scale

```css
:root {
  /* Duration scale */
  --duration-instant: 100ms;
  --duration-fast: 150ms;
  --duration-normal: 250ms;
  --duration-slow: 400ms;
  --duration-slower: 600ms;

  /* Easing scale */
  --ease-default: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-bounce: cubic-bezier(0.175, 0.885, 0.32, 1.275);
  --ease-spring: var(--ease-spring-smooth); /* from spring section */

  /* Distance scale */
  --distance-sm: 4px;
  --distance-md: 12px;
  --distance-lg: 24px;
  --distance-xl: 48px;
}
```

### Motion Principles

1. **Purposeful** — Every animation communicates something (feedback, orientation, focus, continuity)
2. **Quick** — Animations should feel instant. If you notice the animation, it's too long
3. **Natural** — Ease-out for entrances (decelerating), ease-in for exits (accelerating)
4. **Consistent** — Same type of motion uses same timing across the system
5. **Respectful** — Always honor `prefers-reduced-motion`

### Reduced Motion

```css
@media (prefers-reduced-motion: reduce) {
  /* Remove non-essential animations */
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }

  /* Keep essential state changes (opacity for visibility) */
  .toast-enter {
    transition: opacity 0.01ms;
  }
}
```

---

## Performance Optimization

### Composite-Only Properties

Animate only properties that trigger compositing, not layout or paint:

| Safe (Composite) | Avoid (Layout/Paint) |
|-------------------|---------------------|
| `transform` | `width`, `height` |
| `opacity` | `top`, `left`, `right`, `bottom` |
| `filter` | `margin`, `padding` |
| `clip-path` | `border`, `box-shadow` |

```css
/* BAD: triggers layout on every frame */
.slide-in { animation: bad-slide 300ms; }
@keyframes bad-slide {
  from { left: -100%; }
  to { left: 0; }
}

/* GOOD: composite-only transform */
.slide-in { animation: good-slide 300ms; }
@keyframes good-slide {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}
```

### Will-Change Hints

```css
/* Apply will-change BEFORE the animation starts */
.card:hover {
  will-change: transform;
}

.card:hover .card__image {
  transform: scale(1.05);
  transition: transform var(--duration-normal) var(--ease-out);
}

/* Remove will-change after animation completes */
.card:not(:hover) {
  will-change: auto;
}
```

**Rules:**
- Apply `will-change` to elements that WILL animate, not elements that ARE animating
- Don't apply `will-change` to more than a handful of elements
- Remove `will-change` when the animation is complete
- Never use `will-change: all`
