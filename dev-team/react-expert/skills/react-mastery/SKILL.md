---
name: React Mastery
description: This skill should be used when the user asks about "React component", "React hook", "useState", "useEffect", "Server Component", or "error boundary". It covers core React 18+ concepts including hooks, component patterns, error boundaries, Suspense, Server Components, and TypeScript integration with React.
---

### Core Hooks

#### useState and useReducer

To manage simple local state, use `useState`. To manage complex state with multiple sub-values or state transitions that depend on previous state, use `useReducer`.

```typescript
// useState for simple values
const [count, setCount] = useState(0);
const [user, setUser] = useState<User | null>(null);

// useReducer for complex state machines
type Action =
  | { type: 'fetch' }
  | { type: 'success'; data: User[] }
  | { type: 'error'; error: string };

interface State {
  status: 'idle' | 'loading' | 'success' | 'error';
  data: User[];
  error: string | null;
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'fetch':
      return { ...state, status: 'loading', error: null };
    case 'success':
      return { status: 'success', data: action.data, error: null };
    case 'error':
      return { ...state, status: 'error', error: action.error };
  }
}
```

Always type useState generics when the initial value does not fully represent the type (e.g., `useState<User | null>(null)`). Prefer `useReducer` when the next state depends on the previous state in non-trivial ways.

#### useEffect

To synchronize with external systems, use `useEffect`. Never use it for state derivation or event handling that can be done inline.

```typescript
useEffect(() => {
  const controller = new AbortController();

  async function fetchData() {
    try {
      const response = await fetch(`/api/users/${id}`, {
        signal: controller.signal,
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
      if (!controller.signal.aborted) {
        setError(error instanceof Error ? error.message : 'Unknown error');
      }
    }
  }

  fetchData();
  return () => controller.abort();
}, [id]);
```

Always return a cleanup function when subscribing to external sources. Always include AbortController for fetch calls. Lint with `react-hooks/exhaustive-deps` and never suppress without comment.

#### useMemo and useCallback

To skip expensive recalculations, use `useMemo`. To stabilize function references passed to memoized children, use `useCallback`. Do not wrap every value; profile first.

```typescript
const sortedItems = useMemo(
  () => items.toSorted((a, b) => a.name.localeCompare(b.name)),
  [items]
);

const handleSubmit = useCallback(
  async (values: FormValues) => {
    await api.submitForm(values);
    onSuccess();
  },
  [onSuccess]
);
```

#### useRef

To hold mutable values that do not trigger re-renders, or to reference DOM elements, use `useRef`.

```typescript
const inputRef = useRef<HTMLInputElement>(null);
const intervalIdRef = useRef<ReturnType<typeof setInterval> | null>(null);
```

Always type refs with the correct element or value type. Use `null` as the initial value for DOM refs.

#### Custom Hooks

To extract and reuse stateful logic, create custom hooks prefixed with `use`.

```typescript
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

Keep custom hooks focused on a single concern. Compose multiple hooks rather than building monolithic ones.

### Component Patterns

#### Function Components with TypeScript

To define typed components, use explicit interface definitions for props.

```typescript
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  children: React.ReactNode;
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
}

function Button({
  variant,
  size = 'md',
  isLoading = false,
  children,
  onClick,
}: ButtonProps) {
  return (
    <button
      className={clsx(styles.button, styles[variant], styles[size])}
      disabled={isLoading}
      onClick={onClick}
      aria-busy={isLoading}
    >
      {isLoading ? <Spinner size={size} /> : children}
    </button>
  );
}
```

Never use `React.FC` as it adds implicit `children` in older typings and obscures the return type. Prefer destructured props with defaults.

### Error Boundaries

To catch rendering errors and display fallback UI, implement error boundaries as class components (the only remaining class component use case) or use `react-error-boundary`.

```typescript
import { ErrorBoundary, FallbackProps } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary
      FallbackComponent={ErrorFallback}
      onReset={() => queryClient.invalidateQueries()}
    >
      <Dashboard />
    </ErrorBoundary>
  );
}
```

Place error boundaries at strategic points: route level, feature level, and around third-party components. Granular boundaries prevent one failing widget from crashing the entire page.

### Suspense

To declaratively handle loading states, wrap lazy-loaded components or data-fetching components in Suspense boundaries.

```typescript
import { Suspense, lazy } from 'react';

const AnalyticsDashboard = lazy(() => import('./AnalyticsDashboard'));

function App() {
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      <AnalyticsDashboard />
    </Suspense>
  );
}
```

Nest Suspense boundaries to create progressive loading experiences. Combine with error boundaries for complete async UI handling: `ErrorBoundary > Suspense > AsyncComponent`.

### Server Components vs Client Components

To decide between Server and Client Components, follow these rules:

**Default to Server Components** when the component:
- Fetches data
- Accesses backend resources directly
- Contains sensitive logic (API keys, tokens)
- Renders static or infrequently changing content
- Has large dependencies that should stay off the client bundle

**Use Client Components** (`"use client"`) when the component:
- Uses hooks (useState, useEffect, etc.)
- Needs browser APIs (window, localStorage, IntersectionObserver)
- Adds event listeners (onClick, onChange, etc.)
- Uses context providers or consumers
- Requires lifecycle side effects

```typescript
// Server Component (default in Next.js App Router)
async function UserProfile({ userId }: { userId: string }) {
  const user = await db.user.findUnique({ where: { id: userId } });
  return (
    <div>
      <h1>{user.name}</h1>
      <UserActions userId={userId} /> {/* Client Component */}
    </div>
  );
}

// Client Component
'use client';

import { useState } from 'react';

function UserActions({ userId }: { userId: string }) {
  const [isFollowing, setIsFollowing] = useState(false);
  // interactive logic here
}
```

Push `"use client"` boundaries as far down the component tree as possible. Pass Server Component output as `children` to Client Components to keep the server-rendered content on the server.

### TypeScript with React

To maximize type safety, follow these conventions:

- Use `interface` for props (extendable) and `type` for unions/intersections
- Type event handlers explicitly: `React.ChangeEvent<HTMLInputElement>`
- Use generics for reusable components: `function List<T>({ items }: { items: T[] })`
- Type context with explicit interfaces and use a custom hook that throws when context is missing
- Use `satisfies` for config objects to get both type checking and type narrowing
- Use discriminated unions for component variants

```typescript
interface BaseProps {
  className?: string;
  children: React.ReactNode;
}

interface LinkButtonProps extends BaseProps {
  as: 'link';
  href: string;
}

interface ButtonElementProps extends BaseProps {
  as?: 'button';
  onClick: () => void;
}

type PolymorphicButtonProps = LinkButtonProps | ButtonElementProps;

function PolymorphicButton(props: PolymorphicButtonProps) {
  if (props.as === 'link') {
    return <a href={props.href} className={props.className}>{props.children}</a>;
  }
  return <button onClick={props.onClick} className={props.className}>{props.children}</button>;
}
```

## References

- [solid-patterns.md](references/solid-patterns.md) - React-specific SOLID principles with TypeScript examples covering SRP, OCP, LSP, ISP, and DIP applied to components, hooks, and context.
- [best-practices.md](references/best-practices.md) - React 18+ concurrent features, Server Components, streaming SSR, TypeScript strict mode, accessibility standards, and performance profiling techniques.
