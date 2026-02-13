# React Project Structure

## Feature-Based vs Layer-Based

### Layer-Based (Avoid for Large Projects)

Layer-based organization groups files by type. It works for small projects but breaks down as the application grows.

```
src/
  components/
    Button.tsx
    ProductCard.tsx
    UserAvatar.tsx
    CheckoutForm.tsx      # 200 files in one folder
  hooks/
    useProducts.ts
    useAuth.ts
    useCheckout.ts
  utils/
    formatPrice.ts
    validateEmail.ts
  types/
    product.ts
    user.ts
```

Problems with layer-based structure at scale:
- Hundreds of files per folder, hard to navigate
- No clear ownership: who maintains `ProductCard` vs `CheckoutForm`?
- Changes to one feature touch many folders
- Circular dependency risk between layers
- No encapsulation: everything is equally accessible

### Feature-Based (Recommended)

Feature-based organization groups all code related to a domain concept together. Each feature is a self-contained module.

```
src/
  features/
    auth/
      components/
        LoginForm.tsx
        RegisterForm.tsx
        ForgotPasswordForm.tsx
        AuthGuard.tsx
      hooks/
        useAuth.ts
        useSession.ts
      api/
        auth.api.ts
      types/
        auth.types.ts
      utils/
        token.ts
      __tests__/
        LoginForm.test.tsx
        useAuth.test.ts
      index.ts
    products/
      components/
        ProductCard.tsx
        ProductGrid.tsx
        ProductDetail.tsx
        ProductFilters.tsx
        ProductSearch.tsx
      hooks/
        useProducts.ts
        useProductSearch.ts
        useProductFilters.ts
      api/
        products.api.ts
      types/
        product.types.ts
      constants/
        product.constants.ts
      __tests__/
        ProductCard.test.tsx
        ProductGrid.test.tsx
      index.ts
    checkout/
      components/
        Cart.tsx
        ShippingForm.tsx
        PaymentForm.tsx
        OrderSummary.tsx
        CheckoutWizard.tsx
      hooks/
        useCart.ts
        useCheckout.ts
      api/
        checkout.api.ts
      types/
        checkout.types.ts
      __tests__/
      index.ts
  shared/
    components/
      Button/
        Button.tsx
        Button.test.tsx
        Button.module.css
        index.ts
      Input/
      Modal/
      Toast/
    hooks/
      useDebounce.ts
      useMediaQuery.ts
      useLocalStorage.ts
      useClickOutside.ts
    utils/
      formatters.ts
      validators.ts
      cn.ts
    types/
      common.types.ts
    constants/
      routes.ts
      config.ts
  app/
    routes.tsx
    providers.tsx
    layout/
      RootLayout.tsx
      DashboardLayout.tsx
      AuthLayout.tsx
```

### Feature Module Rules

1. **Public API**: Each feature exports only what other features need through its `index.ts`
2. **No deep imports**: Import from `@/features/products`, never from `@/features/products/hooks/useProducts`
3. **One-way dependencies**: Features can depend on `shared/` but not on other features directly. If two features need to communicate, lift shared logic to `shared/` or use events/state management
4. **Colocation**: Tests, styles, types, and constants live next to the code they relate to

```typescript
// features/products/index.ts — Public API
export { ProductCard } from './components/ProductCard';
export { ProductGrid } from './components/ProductGrid';
export { ProductDetail } from './components/ProductDetail';
export { useProducts } from './hooks/useProducts';
export { useProductSearch } from './hooks/useProductSearch';
export type { Product, ProductFilters, ProductCategory } from './types/product.types';

// GOOD: Other features import from the public API
import { ProductCard, useProducts } from '@/features/products';

// BAD: Deep import violates module boundary
import { ProductCard } from '@/features/products/components/ProductCard';
```

## Module Boundaries

### Enforcing Boundaries

Use ESLint rules to enforce import restrictions between features.

```javascript
// .eslintrc.js
module.exports = {
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        {
          group: ['@/features/*/components/*', '@/features/*/hooks/*', '@/features/*/api/*'],
          message: 'Import from the feature index file instead: @/features/featureName',
        },
      ],
    }],
  },
};
```

### Cross-Feature Communication

When features need to interact, use one of these patterns:

1. **Shared state store**: Both features read from a shared Zustand store or context
2. **Event bus**: One feature emits events, another subscribes (loose coupling)
3. **Lift to shared**: Move the shared logic to `shared/` where both features can import it
4. **Composition at the page level**: The page component (which knows about multiple features) wires them together

```typescript
// Page-level composition: the page knows about multiple features
import { ProductGrid } from '@/features/products';
import { CartSidebar } from '@/features/checkout';
import { useCartStore } from '@/shared/stores/cart';

function ShopPage() {
  const addToCart = useCartStore((s) => s.addItem);

  return (
    <div className={styles.shop}>
      <ProductGrid onAddToCart={addToCart} />
      <CartSidebar />
    </div>
  );
}
```

## Shared Component Libraries

### Component Directory Structure

Each shared component gets its own directory with collocated tests, styles, and types.

```
shared/
  components/
    Button/
      Button.tsx
      Button.test.tsx
      Button.module.css
      Button.stories.tsx       # Storybook story
      button.variants.ts       # cva/tailwind-variants definitions
      index.ts                 # Re-exports Button
    DataTable/
      DataTable.tsx
      DataTableHeader.tsx
      DataTableBody.tsx
      DataTablePagination.tsx
      DataTable.test.tsx
      DataTable.module.css
      index.ts                 # Re-exports all sub-components
```

### Design System Tokens

Centralize design decisions in tokens, not scattered across components.

```typescript
// shared/tokens/spacing.ts
export const spacing = {
  xs: '0.25rem',
  sm: '0.5rem',
  md: '1rem',
  lg: '1.5rem',
  xl: '2rem',
  '2xl': '3rem',
} as const;

// shared/tokens/colors.ts
export const colors = {
  primary: { 50: '#eff6ff', 500: '#3b82f6', 900: '#1e3a5f' },
  neutral: { 50: '#fafafa', 500: '#737373', 900: '#171717' },
  danger: { 50: '#fef2f2', 500: '#ef4444', 900: '#7f1d1d' },
} as const;
```

## Monorepo Patterns

### Turborepo / Nx Structure

```
packages/
  ui/                        # Shared component library
    src/
      Button/
      Input/
      Modal/
    package.json
  utils/                     # Shared utilities
    src/
      formatters.ts
      validators.ts
    package.json
  config/                    # Shared configs (ESLint, TypeScript, Tailwind)
    eslint-config/
    tsconfig/
    tailwind-config/
apps/
  web/                       # Main web application
    src/
      features/
      app/
    package.json
  admin/                     # Admin dashboard
    src/
    package.json
  docs/                      # Documentation site
    src/
    package.json
```

### Internal Package Conventions

```json
// packages/ui/package.json
{
  "name": "@acme/ui",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./Button": "./src/Button/index.ts",
    "./Input": "./src/Input/index.ts"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

Monorepo benefits:
- Share components and utilities across applications
- Consistent tooling and configuration
- Atomic commits across packages
- Independent versioning and deployment when needed

## Lazy Loading

### Route-Based Lazy Loading

The most impactful code splitting happens at the route level. Each route becomes its own chunk.

```typescript
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Each route is a separate chunk
const Home = lazy(() => import('@/pages/Home'));
const Products = lazy(() => import('@/pages/Products'));
const ProductDetail = lazy(() => import('@/pages/ProductDetail'));
const Checkout = lazy(() => import('@/pages/Checkout'));
const Account = lazy(() => import('@/pages/Account'));
const Admin = lazy(() => import('@/pages/Admin'));

function AppRoutes() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/products" element={<Products />} />
        <Route path="/products/:id" element={<ProductDetail />} />
        <Route path="/checkout" element={<Checkout />} />
        <Route path="/account/*" element={<Account />} />
        <Route path="/admin/*" element={<Admin />} />
      </Routes>
    </Suspense>
  );
}
```

### Component-Level Lazy Loading

Load heavy components on demand when they enter the viewport or when the user triggers an action.

```typescript
import { lazy, Suspense, useState } from 'react';

// Load on user action (button click)
const HeavyEditor = lazy(() => import('./HeavyEditor'));

function EditorSection() {
  const [showEditor, setShowEditor] = useState(false);

  return (
    <div>
      {!showEditor && (
        <button onClick={() => setShowEditor(true)}>Open Editor</button>
      )}
      {showEditor && (
        <Suspense fallback={<EditorSkeleton />}>
          <HeavyEditor />
        </Suspense>
      )}
    </div>
  );
}
```

### Intersection Observer Loading

```typescript
function LazySection({ importFn, fallback }: LazySectionProps) {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { rootMargin: '200px' } // Start loading 200px before visible
    );

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  const LazyComponent = useMemo(
    () => (isVisible ? lazy(importFn) : null),
    [isVisible, importFn]
  );

  return (
    <div ref={ref}>
      {LazyComponent ? (
        <Suspense fallback={fallback}>
          <LazyComponent />
        </Suspense>
      ) : (
        fallback
      )}
    </div>
  );
}

// Usage
<LazySection
  importFn={() => import('./HeavyChartSection')}
  fallback={<ChartSkeleton />}
/>
```

## Code Splitting Strategies

### Named Exports with Lazy Loading

`React.lazy` requires a default export. For named exports, use a re-export wrapper.

```typescript
// Direct: only works with default exports
const Chart = lazy(() => import('./Chart'));

// Named export workaround
const Chart = lazy(() =>
  import('./Chart').then((module) => ({ default: module.Chart }))
);
```

### Prefetching

Prefetch the next likely route when the user hovers over a link or when the current page finishes loading.

```typescript
function prefetch(importFn: () => Promise<unknown>) {
  // Trigger the import to start downloading the chunk
  importFn();
}

function NavLink({ to, importFn, children }: NavLinkProps) {
  return (
    <Link
      to={to}
      onMouseEnter={() => prefetch(importFn)}
      onFocus={() => prefetch(importFn)}
    >
      {children}
    </Link>
  );
}

// Usage
<NavLink to="/products" importFn={() => import('@/pages/Products')}>
  Products
</NavLink>
```

### Bundle Analysis

Regularly analyze bundle composition to find optimization opportunities.

```bash
# Next.js
ANALYZE=true next build

# Vite
npx vite-bundle-visualizer

# Generic
npx source-map-explorer dist/assets/*.js
```

Look for:
- Chunks larger than 200 KB (candidates for further splitting)
- Duplicate dependencies across chunks
- Libraries that could be replaced with lighter alternatives
- Code that should be lazy loaded but is in the initial bundle
