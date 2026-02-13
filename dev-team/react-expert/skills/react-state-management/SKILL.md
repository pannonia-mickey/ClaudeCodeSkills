---
name: React State Management
description: This skill should be used when the user asks about "React state", "Redux", "Zustand", "Context API", "TanStack Query", or "global state". It covers choosing a state management approach, implementing state patterns, migrating between state libraries, and understanding the tradeoffs between local state, Context API, and third-party state management solutions in React applications.
---

### State Categorization

Before choosing a tool, categorize the state. Different categories demand different solutions.

**Local UI State**: Form inputs, toggle states, modal visibility, accordion expansion, active tab. Managed with `useState` or `useReducer`. Lives in the component that owns it.

**Global Client State**: Theme preferences, authenticated user, sidebar collapsed state, feature flags, locale. Shared across many components. Managed with Context API, Zustand, Jotai, or Redux Toolkit.

**Server/Cache State**: API responses, paginated lists, real-time data feeds, user profiles fetched from the backend. Managed with TanStack Query, SWR, or Apollo Client. These tools handle caching, background refetching, stale-while-revalidate, optimistic updates, and deduplication.

**URL State**: Current route, search parameters, pagination page number, active filters. Managed by the router (React Router, Next.js routing). Derive from the URL rather than duplicating in component state.

**Form State**: Multi-field forms with validation, dirty tracking, and submission. Managed by React Hook Form, Formik, or native `useReducer` for simple cases.

### Decision Tree

To choose the right state management approach, follow this decision tree:

1. Is the state used by only one component? Use `useState` or `useReducer`.
2. Is the state shared between a parent and a few children? Lift state up to the nearest common ancestor.
3. Is the state fetched from a server? Use TanStack Query or SWR.
4. Is the state URL-representable (filters, pagination, search)? Derive from URL parameters.
5. Is the state shared across many distant components but changes infrequently? Use Context API.
6. Is the state shared across many components and changes frequently? Use Zustand, Jotai, or Redux Toolkit.

### useState and useReducer

To manage local state, start with `useState` for simple values and switch to `useReducer` when state transitions become complex.

```typescript
// useState: simple toggle
const [isOpen, setIsOpen] = useState(false);

// useReducer: complex state machine
interface FormState {
  values: Record<string, string>;
  errors: Record<string, string>;
  touched: Record<string, boolean>;
  isSubmitting: boolean;
}

type FormAction =
  | { type: 'SET_FIELD'; field: string; value: string }
  | { type: 'SET_ERROR'; field: string; error: string }
  | { type: 'TOUCH_FIELD'; field: string }
  | { type: 'SUBMIT_START' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_FAILURE'; errors: Record<string, string> }
  | { type: 'RESET' };

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: '' },
      };
    case 'SET_ERROR':
      return {
        ...state,
        errors: { ...state.errors, [action.field]: action.error },
      };
    case 'TOUCH_FIELD':
      return {
        ...state,
        touched: { ...state.touched, [action.field]: true },
      };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true };
    case 'SUBMIT_SUCCESS':
      return { ...initialFormState };
    case 'SUBMIT_FAILURE':
      return { ...state, isSubmitting: false, errors: action.errors };
    case 'RESET':
      return { ...initialFormState };
  }
}
```

Switch from `useState` to `useReducer` when: there are more than three related state variables, the next state depends on the previous state, or you need to handle multiple action types.

### Context API

To share state across a subtree without prop drilling, use Context. Create a typed context with a custom hook that throws when used outside its provider.

```typescript
interface NotificationContextValue {
  notifications: Notification[];
  addNotification: (notification: Omit<Notification, 'id'>) => void;
  dismissNotification: (id: string) => void;
  clearAll: () => void;
}

const NotificationContext = createContext<NotificationContextValue | null>(null);

export function useNotifications(): NotificationContextValue {
  const context = useContext(NotificationContext);
  if (!context) {
    throw new Error('useNotifications must be used within NotificationProvider');
  }
  return context;
}

export function NotificationProvider({ children }: { children: React.ReactNode }) {
  const [notifications, dispatch] = useReducer(notificationReducer, []);

  const value = useMemo<NotificationContextValue>(() => ({
    notifications,
    addNotification: (notification) =>
      dispatch({ type: 'ADD', notification: { ...notification, id: crypto.randomUUID() } }),
    dismissNotification: (id) =>
      dispatch({ type: 'DISMISS', id }),
    clearAll: () =>
      dispatch({ type: 'CLEAR' }),
  }), [notifications]);

  return (
    <NotificationContext.Provider value={value}>
      {children}
    </NotificationContext.Provider>
  );
}
```

#### Context Performance Considerations

Context has a re-render cost: every consumer re-renders when the context value changes. Mitigate this by:

1. **Splitting contexts**: Separate frequently changing values from stable ones
2. **Memoizing the value**: Wrap the value object in `useMemo`
3. **Splitting read and dispatch**: Create separate contexts for state and actions

```typescript
// Split state and dispatch into separate contexts
const StateContext = createContext<State | null>(null);
const DispatchContext = createContext<Dispatch<Action> | null>(null);

// Components that only dispatch actions do not re-render when state changes
function AddButton() {
  const dispatch = useContext(DispatchContext)!;
  return <button onClick={() => dispatch({ type: 'ADD' })}>Add</button>;
}
```

### Library Comparison Overview

| Library | Best For | Bundle Size | Learning Curve |
|---|---|---|---|
| Context API | Low-frequency global state | 0 KB (built-in) | Low |
| Zustand | Simple global state | ~1 KB | Low |
| Jotai | Atomic state, derived state | ~2 KB | Low-Medium |
| Redux Toolkit | Complex enterprise apps | ~11 KB | Medium |
| TanStack Query | Server/cache state | ~13 KB | Medium |
| XState | Complex state machines | ~15 KB | High |

**Zustand** is the best default choice for global client state. It is tiny, has almost no boilerplate, works outside React components, and supports middleware (persist, devtools, immer).

**TanStack Query** is the best choice for server state. Do not store API responses in Redux or Zustand; let a dedicated server-state library handle caching, refetching, and synchronization.

**Redux Toolkit** is justified in large applications with complex state interactions, time-travel debugging needs, or teams already familiar with the Redux ecosystem.

**Jotai** excels when state is highly granular and derived: atoms compose together to create complex derived state with minimal re-renders.

### Zustand Quick Start

```typescript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface CartStore {
  items: CartItem[];
  addItem: (product: Product, quantity: number) => void;
  removeItem: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  clearCart: () => void;
  totalItems: () => number;
  totalPrice: () => number;
}

export const useCartStore = create<CartStore>()(
  devtools(
    persist(
      immer((set, get) => ({
        items: [],
        addItem: (product, quantity) =>
          set((state) => {
            const existing = state.items.find((i) => i.productId === product.id);
            if (existing) {
              existing.quantity += quantity;
            } else {
              state.items.push({
                productId: product.id,
                name: product.name,
                price: product.price,
                quantity,
              });
            }
          }),
        removeItem: (productId) =>
          set((state) => {
            state.items = state.items.filter((i) => i.productId !== productId);
          }),
        updateQuantity: (productId, quantity) =>
          set((state) => {
            const item = state.items.find((i) => i.productId === productId);
            if (item) item.quantity = quantity;
          }),
        clearCart: () => set({ items: [] }),
        totalItems: () => get().items.reduce((sum, i) => sum + i.quantity, 0),
        totalPrice: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
      })),
      { name: 'cart-storage' }
    )
  )
);
```

### TanStack Query Quick Start

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Query: fetch data
function useProducts(category: string) {
  return useQuery({
    queryKey: ['products', category],
    queryFn: () => api.getProducts(category),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

// Mutation: modify data with optimistic update
function useUpdateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (product: UpdateProductInput) => api.updateProduct(product),
    onMutate: async (updatedProduct) => {
      await queryClient.cancelQueries({ queryKey: ['products'] });
      const previous = queryClient.getQueryData(['products']);
      queryClient.setQueryData(['products'], (old: Product[]) =>
        old.map((p) => (p.id === updatedProduct.id ? { ...p, ...updatedProduct } : p))
      );
      return { previous };
    },
    onError: (_err, _variables, context) => {
      queryClient.setQueryData(['products'], context?.previous);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

## References

- [state-patterns.md](references/state-patterns.md) - Advanced state patterns including local vs global vs server state boundaries, state machines with XState, optimistic updates, cache invalidation strategies, derived state, and state normalization.
- [state-comparison.md](references/state-comparison.md) - Detailed comparison of Redux Toolkit, Zustand, Jotai, and TanStack Query with use-case recommendations, migration paths, and performance characteristics.
