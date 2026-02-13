---
name: React Component Architecture
description: This skill should be used when the user asks about "React project structure", "component architecture", "compound component", or "atomic design". It covers structuring React applications, designing component hierarchies, implementing advanced component patterns like compound components or render props, organizing features in a scalable project structure, and implementing code splitting and lazy loading strategies.
---

### Atomic Design in React

Atomic design provides a mental model for building component libraries from small, composable pieces. The hierarchy from smallest to largest: atoms, molecules, organisms, templates, pages.

**Atoms** are the smallest building blocks with no dependencies on other components. They map closely to native HTML elements with added styling and type safety.

```typescript
// atoms/Button.tsx
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant: 'primary' | 'secondary' | 'ghost';
  size: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
}

function Button({ variant, size, isLoading, children, ...rest }: ButtonProps) {
  return (
    <button
      className={clsx(styles.base, styles[variant], styles[size])}
      disabled={isLoading || rest.disabled}
      aria-busy={isLoading}
      {...rest}
    >
      {isLoading ? <Spinner size={size} /> : children}
    </button>
  );
}
```

**Molecules** combine atoms into small functional units. A search input (input atom + button atom + icon atom) or a form field (label atom + input atom + error message atom).

```typescript
// molecules/FormField.tsx
interface FormFieldProps {
  label: string;
  error?: string;
  required?: boolean;
  children: React.ReactElement;
}

function FormField({ label, error, required, children }: FormFieldProps) {
  const id = useId();
  return (
    <div className={styles.field}>
      <Label htmlFor={id} required={required}>{label}</Label>
      {cloneElement(children, { id, 'aria-describedby': error ? `${id}-error` : undefined })}
      {error && <ErrorMessage id={`${id}-error`}>{error}</ErrorMessage>}
    </div>
  );
}
```

**Organisms** are complex UI sections composed of molecules and atoms: navigation bars, product cards with actions, comment threads, data table headers.

**Templates** define page layouts without real data. They accept children or named slots and handle responsive grid placement.

**Pages** are template instances filled with real data, connected to state management and routing.

Apply atomic design flexibly. Not every project needs all five levels. The key insight is building from small, tested units toward larger compositions.

### Compound Components

Compound components share implicit state through context, giving consumers full control over rendering order and composition while the parent manages shared logic.

```typescript
// Compound component pattern
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const context = useContext(TabsContext);
  if (!context) throw new Error('Tabs compound components must be used within <Tabs>');
  return context;
}

function Tabs({ defaultTab, children, onChange }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  const handleTabChange = useCallback((id: string) => {
    setActiveTab(id);
    onChange?.(id);
  }, [onChange]);

  const value = useMemo(
    () => ({ activeTab, setActiveTab: handleTabChange }),
    [activeTab, handleTabChange]
  );

  return (
    <TabsContext.Provider value={value}>
      <div className={styles.tabs}>{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: React.ReactNode }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeTab, setActiveTab } = useTabsContext();
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      tabIndex={activeTab === id ? 0 : -1}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeTab } = useTabsContext();
  if (activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

// Attach sub-components
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Consumer has full control over composition
function SettingsPage() {
  return (
    <Tabs defaultTab="general">
      <Tabs.List>
        <Tabs.Tab id="general">General</Tabs.Tab>
        <Tabs.Tab id="security">Security</Tabs.Tab>
        <Tabs.Tab id="notifications">Notifications</Tabs.Tab>
      </Tabs.List>
      <Tabs.Panel id="general"><GeneralSettings /></Tabs.Panel>
      <Tabs.Panel id="security"><SecuritySettings /></Tabs.Panel>
      <Tabs.Panel id="notifications"><NotificationSettings /></Tabs.Panel>
    </Tabs>
  );
}
```

### Render Props

To share behavior while giving consumers full control over rendering, use the render props pattern. This is less common since hooks, but still valuable for components that manage complex UI state like virtualization or drag-and-drop.

```typescript
interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  containerHeight: number;
  renderItem: (item: T, index: number, style: React.CSSProperties) => React.ReactNode;
}

function VirtualList<T>({ items, itemHeight, containerHeight, renderItem }: VirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);

  const startIndex = Math.floor(scrollTop / itemHeight);
  const endIndex = Math.min(
    startIndex + Math.ceil(containerHeight / itemHeight) + 1,
    items.length
  );
  const visibleItems = items.slice(startIndex, endIndex);

  return (
    <div
      style={{ height: containerHeight, overflow: 'auto' }}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ height: items.length * itemHeight, position: 'relative' }}>
        {visibleItems.map((item, i) =>
          renderItem(item, startIndex + i, {
            position: 'absolute',
            top: (startIndex + i) * itemHeight,
            height: itemHeight,
            width: '100%',
          })
        )}
      </div>
    </div>
  );
}
```

### Feature-Based Structure

Organize code by feature (domain), not by type (components, hooks, utils). Each feature is a self-contained module with its own components, hooks, types, and tests.

```
src/
  features/
    products/
      components/
        ProductCard.tsx
        ProductGrid.tsx
        ProductFilters.tsx
      hooks/
        useProducts.ts
        useProductFilters.ts
      api/
        products.api.ts
      types/
        product.types.ts
      __tests__/
        ProductCard.test.tsx
        useProducts.test.ts
      index.ts           # Public API (barrel file)
    checkout/
      components/
      hooks/
      api/
      types/
      index.ts
  shared/
    components/          # Cross-feature reusable components
    hooks/               # Cross-feature reusable hooks
    utils/               # Pure utility functions
    types/               # Shared type definitions
  app/
    routes/              # Route definitions
    providers/           # App-level providers
    layout/              # Layout components
```

Each feature exports a public API through its `index.ts`. Other features import only from this barrel file, never from internal paths. This enforces module boundaries and makes refactoring safe.

```typescript
// features/products/index.ts — Public API
export { ProductCard } from './components/ProductCard';
export { ProductGrid } from './components/ProductGrid';
export { useProducts } from './hooks/useProducts';
export type { Product, ProductFilters } from './types/product.types';
```

### Code Splitting

To reduce the initial bundle, split code at route boundaries and for heavy components.

```typescript
// Route-level splitting with React.lazy
import { lazy, Suspense } from 'react';

const ProductsPage = lazy(() => import('./pages/ProductsPage'));
const CheckoutPage = lazy(() => import('./pages/CheckoutPage'));
const AdminDashboard = lazy(() => import('./pages/AdminDashboard'));

function AppRoutes() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/products" element={<ProductsPage />} />
        <Route path="/checkout" element={<CheckoutPage />} />
        <Route path="/admin/*" element={<AdminDashboard />} />
      </Routes>
    </Suspense>
  );
}
```

```typescript
// Component-level splitting for heavy dependencies
import dynamic from 'next/dynamic';

const MarkdownEditor = dynamic(() => import('./MarkdownEditor'), {
  loading: () => <EditorSkeleton />,
  ssr: false,
});

const ChartDashboard = dynamic(() => import('./ChartDashboard'), {
  loading: () => <ChartSkeleton />,
});
```

Split at these boundaries:
- Route level: each page is a separate chunk
- Modal content: heavy modals load on demand
- Below-the-fold content: load when scrolled into view
- Admin/authenticated sections: do not load admin UI for regular users
- Heavy third-party libraries: chart libraries, rich text editors, PDF viewers

### Named Exports and Barrel Files

Use named exports for components to enable tree shaking and consistent imports.

```typescript
// GOOD: Named export
export function ProductCard(props: ProductCardProps) { /* ... */ }

// BAD: Default export (harder to rename consistently, worse tree shaking)
export default function ProductCard(props: ProductCardProps) { /* ... */ }
```

Barrel files (`index.ts`) define the public API of a module. Keep them thin to avoid circular dependencies and ensure bundlers can tree-shake effectively.

```typescript
// features/products/index.ts
export { ProductCard } from './components/ProductCard';
export { ProductGrid } from './components/ProductGrid';
export { useProducts } from './hooks/useProducts';
// Do NOT re-export internal utilities or private components
```

## References

- [component-patterns.md](references/component-patterns.md) - Advanced component patterns including compound components, controlled vs uncontrolled components, polymorphic components (as prop), headless UI, render props, and the slot pattern.
- [project-structure.md](references/project-structure.md) - Detailed project organization strategies covering feature-based vs layer-based structures, module boundaries, shared component libraries, monorepo patterns, lazy loading, and code splitting techniques.
