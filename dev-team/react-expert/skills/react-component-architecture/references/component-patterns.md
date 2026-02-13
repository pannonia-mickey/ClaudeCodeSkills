# React Component Patterns

## Compound Components

Compound components share implicit state through React context, allowing consumers to compose sub-components in any order while the parent manages shared behavior.

### Basic Pattern

```typescript
import { createContext, useContext, useState, useCallback, useMemo } from 'react';

// 1. Define context type
interface AccordionContextValue {
  expandedItems: Set<string>;
  toggleItem: (id: string) => void;
  allowMultiple: boolean;
}

// 2. Create context
const AccordionContext = createContext<AccordionContextValue | null>(null);

function useAccordionContext() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('Accordion sub-components must be used within <Accordion>');
  return ctx;
}

// 3. Root component manages state
interface AccordionProps {
  allowMultiple?: boolean;
  defaultExpanded?: string[];
  children: React.ReactNode;
}

function Accordion({ allowMultiple = false, defaultExpanded = [], children }: AccordionProps) {
  const [expandedItems, setExpandedItems] = useState<Set<string>>(new Set(defaultExpanded));

  const toggleItem = useCallback((id: string) => {
    setExpandedItems((prev) => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) next.clear();
        next.add(id);
      }
      return next;
    });
  }, [allowMultiple]);

  const value = useMemo(
    () => ({ expandedItems, toggleItem, allowMultiple }),
    [expandedItems, toggleItem, allowMultiple]
  );

  return (
    <AccordionContext.Provider value={value}>
      <div role="region">{children}</div>
    </AccordionContext.Provider>
  );
}

// 4. Sub-components consume context
function AccordionItem({ id, children }: { id: string; children: React.ReactNode }) {
  return <div data-accordion-item={id}>{children}</div>;
}

function AccordionTrigger({ id, children }: { id: string; children: React.ReactNode }) {
  const { expandedItems, toggleItem } = useAccordionContext();
  const isExpanded = expandedItems.has(id);

  return (
    <button
      aria-expanded={isExpanded}
      aria-controls={`panel-${id}`}
      onClick={() => toggleItem(id)}
    >
      {children}
      <ChevronIcon direction={isExpanded ? 'up' : 'down'} />
    </button>
  );
}

function AccordionContent({ id, children }: { id: string; children: React.ReactNode }) {
  const { expandedItems } = useAccordionContext();
  const isExpanded = expandedItems.has(id);

  return (
    <div
      id={`panel-${id}`}
      role="region"
      hidden={!isExpanded}
      aria-hidden={!isExpanded}
    >
      {isExpanded && children}
    </div>
  );
}

// 5. Attach to root
Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;
```

### Consumer Usage

```typescript
<Accordion allowMultiple defaultExpanded={['faq-1']}>
  <Accordion.Item id="faq-1">
    <Accordion.Trigger id="faq-1">What is your return policy?</Accordion.Trigger>
    <Accordion.Content id="faq-1">
      <p>You can return items within 30 days of purchase.</p>
    </Accordion.Content>
  </Accordion.Item>
  <Accordion.Item id="faq-2">
    <Accordion.Trigger id="faq-2">How long does shipping take?</Accordion.Trigger>
    <Accordion.Content id="faq-2">
      <p>Standard shipping takes 5-7 business days.</p>
    </Accordion.Content>
  </Accordion.Item>
</Accordion>
```

## Controlled vs Uncontrolled Components

### Uncontrolled

The component manages its own state internally. The consumer provides defaults and callbacks.

```typescript
interface UncontrolledToggleProps {
  defaultChecked?: boolean;
  onChange?: (checked: boolean) => void;
  label: string;
}

function UncontrolledToggle({ defaultChecked = false, onChange, label }: UncontrolledToggleProps) {
  const [checked, setChecked] = useState(defaultChecked);

  function handleChange() {
    const next = !checked;
    setChecked(next);
    onChange?.(next);
  }

  return (
    <label>
      <input type="checkbox" checked={checked} onChange={handleChange} />
      {label}
    </label>
  );
}
```

### Controlled

The consumer owns the state entirely and passes it via props.

```typescript
interface ControlledToggleProps {
  checked: boolean;
  onChange: (checked: boolean) => void;
  label: string;
}

function ControlledToggle({ checked, onChange, label }: ControlledToggleProps) {
  return (
    <label>
      <input type="checkbox" checked={checked} onChange={() => onChange(!checked)} />
      {label}
    </label>
  );
}
```

### Hybrid (Controlled or Uncontrolled)

Support both modes. If `value` is provided, the component is controlled. Otherwise, it manages its own state.

```typescript
interface ToggleProps {
  checked?: boolean;
  defaultChecked?: boolean;
  onChange?: (checked: boolean) => void;
  label: string;
}

function Toggle({ checked: controlledChecked, defaultChecked = false, onChange, label }: ToggleProps) {
  const [internalChecked, setInternalChecked] = useState(defaultChecked);
  const isControlled = controlledChecked !== undefined;
  const checked = isControlled ? controlledChecked : internalChecked;

  function handleChange() {
    const next = !checked;
    if (!isControlled) {
      setInternalChecked(next);
    }
    onChange?.(next);
  }

  return (
    <label>
      <input type="checkbox" checked={checked} onChange={handleChange} />
      {label}
    </label>
  );
}
```

## Polymorphic Components (as prop)

Polymorphic components render as different HTML elements or React components based on an `as` prop, while preserving type-safe props for the rendered element.

```typescript
type PolymorphicProps<E extends React.ElementType, P = {}> = P &
  Omit<React.ComponentPropsWithoutRef<E>, keyof P | 'as'> & {
    as?: E;
  };

type TextProps<E extends React.ElementType = 'span'> = PolymorphicProps<E, {
  size?: 'xs' | 'sm' | 'md' | 'lg' | 'xl';
  weight?: 'normal' | 'medium' | 'bold';
  color?: 'primary' | 'secondary' | 'muted';
}>;

function Text<E extends React.ElementType = 'span'>({
  as,
  size = 'md',
  weight = 'normal',
  color = 'primary',
  className,
  children,
  ...rest
}: TextProps<E>) {
  const Component = as || 'span';

  return (
    <Component
      className={clsx(styles.text, styles[size], styles[weight], styles[color], className)}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Usage: renders as different elements with correct type checking
<Text>Default span</Text>
<Text as="h1" size="xl" weight="bold">Page Title</Text>
<Text as="p" color="muted">Description paragraph</Text>
<Text as="a" href="/about" color="primary">Link text</Text>
<Text as={Link} to="/about">Router link</Text>
```

The type system ensures that when `as="a"`, the `href` prop is available and typed. When `as="button"`, `onClick` with the correct event type is available.

## Headless UI Components

Headless components provide behavior and accessibility without any visual rendering. The consumer provides all markup and styling.

```typescript
interface UseDropdownReturn {
  isOpen: boolean;
  selectedItem: string | null;
  highlightedIndex: number;
  getToggleProps: () => React.ButtonHTMLAttributes<HTMLButtonElement>;
  getMenuProps: () => React.HTMLAttributes<HTMLUListElement>;
  getItemProps: (item: string, index: number) => React.LiHTMLAttributes<HTMLLIElement>;
}

function useDropdown(items: string[], onSelect?: (item: string) => void): UseDropdownReturn {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedItem, setSelectedItem] = useState<string | null>(null);
  const [highlightedIndex, setHighlightedIndex] = useState(-1);
  const menuId = useId();

  const getToggleProps = useCallback(() => ({
    'aria-haspopup': 'listbox' as const,
    'aria-expanded': isOpen,
    'aria-controls': menuId,
    onClick: () => setIsOpen((prev) => !prev),
    onKeyDown: (e: React.KeyboardEvent) => {
      if (e.key === 'ArrowDown') {
        e.preventDefault();
        setIsOpen(true);
        setHighlightedIndex(0);
      }
    },
  }), [isOpen, menuId]);

  const getMenuProps = useCallback(() => ({
    id: menuId,
    role: 'listbox' as const,
    'aria-activedescendant': highlightedIndex >= 0 ? `${menuId}-item-${highlightedIndex}` : undefined,
  }), [menuId, highlightedIndex]);

  const getItemProps = useCallback((item: string, index: number) => ({
    id: `${menuId}-item-${index}`,
    role: 'option' as const,
    'aria-selected': selectedItem === item,
    onClick: () => {
      setSelectedItem(item);
      setIsOpen(false);
      onSelect?.(item);
    },
    onMouseEnter: () => setHighlightedIndex(index),
  }), [menuId, selectedItem, onSelect]);

  return { isOpen, selectedItem, highlightedIndex, getToggleProps, getMenuProps, getItemProps };
}

// Consumer applies their own styles
function CustomDropdown({ items }: { items: string[] }) {
  const { isOpen, selectedItem, highlightedIndex, getToggleProps, getMenuProps, getItemProps } =
    useDropdown(items);

  return (
    <div className="my-custom-dropdown">
      <button {...getToggleProps()} className="my-trigger">
        {selectedItem ?? 'Select...'}
      </button>
      {isOpen && (
        <ul {...getMenuProps()} className="my-menu">
          {items.map((item, index) => (
            <li
              key={item}
              {...getItemProps(item, index)}
              className={clsx('my-item', index === highlightedIndex && 'highlighted')}
            >
              {item}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

The headless pattern separates behavior (keyboard navigation, ARIA, state) from presentation. Libraries like Radix Primitives and Headless UI follow this pattern.

## Render Props

The render props pattern passes a function as a child or prop that receives data and returns JSX. Use it when the rendering logic varies significantly between consumers.

```typescript
interface MouseTrackerProps {
  children: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    function handleMouseMove(e: MouseEvent) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);

  return <>{children(position)}</>;
}

// Consumer decides how to render
<MouseTracker>
  {({ x, y }) => (
    <div style={{ position: 'absolute', left: x, top: y }}>
      Cursor: {x}, {y}
    </div>
  )}
</MouseTracker>
```

Render props are still useful for components that manage visual state (tooltips, popovers, animations) where a hook cannot manage the DOM relationship.

## Slot Pattern

The slot pattern allows consumers to inject content into specific named areas of a component layout.

```typescript
interface PageLayoutProps {
  header?: React.ReactNode;
  sidebar?: React.ReactNode;
  children: React.ReactNode;
  footer?: React.ReactNode;
}

function PageLayout({ header, sidebar, children, footer }: PageLayoutProps) {
  return (
    <div className={styles.layout}>
      {header && <header className={styles.header}>{header}</header>}
      <div className={styles.body}>
        {sidebar && <aside className={styles.sidebar}>{sidebar}</aside>}
        <main className={styles.main}>{children}</main>
      </div>
      {footer && <footer className={styles.footer}>{footer}</footer>}
    </div>
  );
}

// Consumer fills the slots
<PageLayout
  header={<NavigationBar />}
  sidebar={<DashboardSidebar />}
  footer={<FooterLinks />}
>
  <DashboardContent />
</PageLayout>
```

### Advanced Slots with Type Safety

```typescript
interface DialogProps {
  children: React.ReactNode;
  trigger: React.ReactElement;
  title: React.ReactNode;
  description?: React.ReactNode;
  actions?: React.ReactNode;
}

function Dialog({ children, trigger, title, description, actions }: DialogProps) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      {cloneElement(trigger, { onClick: () => setIsOpen(true) })}
      {isOpen && (
        <div role="dialog" aria-labelledby="dialog-title" aria-modal="true">
          <h2 id="dialog-title">{title}</h2>
          {description && <p>{description}</p>}
          <div>{children}</div>
          {actions && <div className={styles.actions}>{actions}</div>}
          <button onClick={() => setIsOpen(false)} aria-label="Close">X</button>
        </div>
      )}
    </>
  );
}

// Consumer composes with slots
<Dialog
  trigger={<Button variant="danger">Delete Account</Button>}
  title="Confirm Deletion"
  description="This action cannot be undone. All data will be permanently removed."
  actions={
    <>
      <Button variant="ghost" onClick={handleCancel}>Cancel</Button>
      <Button variant="danger" onClick={handleDelete}>Delete</Button>
    </>
  }
>
  <p>Type your email to confirm: <strong>{email}</strong></p>
  <Input value={confirmEmail} onChange={setConfirmEmail} />
</Dialog>
```

Slots are more explicit than children-only patterns. Each slot has a clear purpose, making the component API self-documenting and easier to use correctly.
