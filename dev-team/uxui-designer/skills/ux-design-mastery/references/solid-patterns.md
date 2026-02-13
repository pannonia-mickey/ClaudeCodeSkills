# SOLID Principles for Design Systems

## Single Responsibility — One Component, One Concern

Each component should have exactly one reason to change. When a component handles layout, data fetching, and presentation, changes to any one of those concerns risk breaking the others.

### Anti-Pattern: Monolithic Component

```jsx
// BAD: Button handles layout, theming, analytics, and routing
function ActionButton({ href, onClick, variant, trackEvent, children }) {
  const handleClick = (e) => {
    analytics.track(trackEvent);
    if (href) navigate(href);
    if (onClick) onClick(e);
  };
  return (
    <button className={`btn btn--${variant}`} onClick={handleClick}>
      {children}
    </button>
  );
}
```

### Corrected: Separated Concerns

```jsx
// GOOD: Button only handles rendering and interaction feedback
function Button({ variant = 'primary', onClick, children, ...props }) {
  return (
    <button
      className={`btn btn--${variant}`}
      onClick={onClick}
      {...props}
    >
      {children}
    </button>
  );
}

// Analytics wrapper — separate concern
function TrackedButton({ trackEvent, ...buttonProps }) {
  const handleClick = (e) => {
    analytics.track(trackEvent);
    buttonProps.onClick?.(e);
  };
  return <Button {...buttonProps} onClick={handleClick} />;
}

// Navigation wrapper — separate concern
function LinkButton({ href, ...buttonProps }) {
  const navigate = useNavigate();
  const handleClick = (e) => {
    navigate(href);
    buttonProps.onClick?.(e);
  };
  return <Button {...buttonProps} onClick={handleClick} />;
}
```

### When to Split a Component

Split when a component:
- Has more than 3 distinct visual sections (anatomy elements)
- Renders conditionally based on unrelated conditions
- Accepts props that are only relevant to specific sub-features
- Requires different testing strategies for different parts

---

## Open/Closed — Extend Without Modifying

Design tokens and components should be open for extension but closed for modification. New themes, variants, or brands should be addable without editing base definitions.

### Token Extension Pattern

```css
/* Base tokens — NEVER modify these for a theme */
:root {
  --color-blue-500: #3b82f6;
  --color-blue-600: #2563eb;
  --space-4: 1rem;
  --radius-md: 0.375rem;
}

/* Semantic tokens — map to base tokens, override per theme */
:root {
  --color-primary: var(--color-blue-600);
  --color-primary-hover: var(--color-blue-700);
  --button-radius: var(--radius-md);
}

/* Brand extension — adds new values, doesn't modify base */
[data-theme="brand-b"] {
  --color-primary: var(--color-purple-600);
  --color-primary-hover: var(--color-purple-700);
  --button-radius: var(--radius-full);
}
```

### Component Variant Extension

```css
/* Base button — closed for modification */
.btn {
  padding: var(--button-padding-y) var(--button-padding-x);
  border-radius: var(--button-radius);
  font-size: var(--button-font-size);
  font-weight: var(--button-font-weight);
  transition: background-color var(--duration-fast) var(--ease-default);
}

/* Variants — extend via modifiers, not by editing base */
.btn--primary {
  background: var(--color-primary);
  color: var(--color-text-inverse);
}

.btn--ghost {
  background: transparent;
  color: var(--color-primary);
  border: 1px solid var(--color-primary);
}

/* New variant added without touching existing code */
.btn--danger {
  background: var(--color-error);
  color: var(--color-text-inverse);
}
```

---

## Liskov Substitution — Variant Interchangeability

Any component variant should work wherever the base component is expected. A `<Button variant="danger">` must behave identically to `<Button variant="primary">` in terms of API, sizing, and interaction — only the visual treatment differs.

### Anti-Pattern: Inconsistent Variant Behavior

```jsx
// BAD: Different variants have different APIs
<Button onClick={handleSave}>Save</Button>           // works
<IconButton icon="save" onPress={handleSave} />       // different event name
<LinkButton href="/save" onNavigate={handleSave} />   // different API entirely
```

### Corrected: Consistent Interface

```jsx
// GOOD: All button variants share the same base props
interface ButtonBaseProps {
  onClick?: (e: React.MouseEvent) => void;
  disabled?: boolean;
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
}

// Every variant extends the same base
<Button variant="primary" onClick={handle}>Save</Button>
<Button variant="icon" onClick={handle}><SaveIcon /></Button>
<Button variant="link" onClick={handle}>Save</Button>
```

### Sizing Consistency

All variants of a component must maintain consistent dimensions within the same size tier:

| Size | Height | Min Width | Icon Size | Font Size |
|------|--------|-----------|-----------|-----------|
| sm | 32px | 64px | 16px | 14px |
| md | 40px | 80px | 20px | 16px |
| lg | 48px | 96px | 24px | 18px |

---

## Interface Segregation — Lean Component APIs

Don't force consumers to configure props they don't use. Split complex configurations into composable pieces.

### Anti-Pattern: Overloaded Props

```jsx
// BAD: Card component with 15+ props for every possible configuration
<Card
  title="Order #1234"
  subtitle="Pending"
  image="/order.jpg"
  imagePosition="left"
  imageAspectRatio="16/9"
  badge="New"
  badgeColor="green"
  actions={[{ label: 'View', onClick: handleView }]}
  footer="Last updated 2h ago"
  collapsible={true}
  defaultCollapsed={false}
  onCollapse={handleCollapse}
  draggable={true}
  onDragStart={handleDrag}
  highlighted={true}
/>
```

### Corrected: Compound Components

```jsx
// GOOD: Compose only what you need
<Card>
  <Card.Header>
    <Card.Title>Order #1234</Card.Title>
    <Card.Badge color="green">New</Card.Badge>
  </Card.Header>
  <Card.Body>
    <Card.Image src="/order.jpg" aspectRatio="16/9" />
    <Card.Description>Pending</Card.Description>
  </Card.Body>
  <Card.Footer>
    <Card.Action onClick={handleView}>View</Card.Action>
  </Card.Footer>
</Card>
```

### Rule of Thumb

If a component has more than 7 props, consider whether it should be split into compound components. Props fall into these categories:
- **Essential** (2-4): What every usage needs (children, variant, size)
- **Behavioral** (1-3): Event handlers and controlled state (onClick, onChange, value)
- **Enhancement** (0-2): Optional additions (className, data-testid)

Anything beyond that signals an interface segregation problem.

---

## Dependency Inversion — Tokens Over Hard-Coded Values

Components should depend on semantic tokens (abstractions), not primitive values (concretions). This enables theming, dark mode, and brand customization without editing component code.

### Anti-Pattern: Hard-Coded Values

```css
/* BAD: Direct color and spacing values */
.alert {
  background: #fee2e2;
  border: 1px solid #ef4444;
  color: #991b1b;
  padding: 12px 16px;
  border-radius: 6px;
  margin-bottom: 16px;
}
```

### Corrected: Token Dependencies

```css
/* GOOD: Semantic tokens abstract the implementation */
.alert {
  background: var(--color-bg-error-subtle);
  border: 1px solid var(--color-border-error);
  color: var(--color-text-error);
  padding: var(--space-3) var(--space-4);
  border-radius: var(--radius-md);
  margin-block-end: var(--space-4);
}
```

### Token Dependency Chain

```
Component Code
    ↓ depends on
Semantic Tokens (--color-bg-error-subtle)
    ↓ maps to
Primitive Tokens (--color-red-100)
    ↓ resolves to
Raw Values (#fee2e2)
```

Components must NEVER skip levels. A component should never reference a primitive token directly — always go through semantic tokens. This ensures that a theme change at the semantic level cascades correctly to all components.

### Theming Made Possible by Inversion

```css
/* Light theme — semantic tokens map to light primitives */
:root {
  --color-bg-primary: var(--color-white);
  --color-text-primary: var(--color-gray-900);
  --color-bg-error-subtle: var(--color-red-50);
}

/* Dark theme — remap semantic tokens to dark primitives */
[data-theme="dark"] {
  --color-bg-primary: var(--color-gray-900);
  --color-text-primary: var(--color-gray-50);
  --color-bg-error-subtle: var(--color-red-950);
}
```

Zero component code changes. The inversion of dependencies makes theming a pure token-mapping exercise.
