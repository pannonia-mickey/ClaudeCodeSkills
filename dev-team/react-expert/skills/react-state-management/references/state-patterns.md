# React State Patterns

## Local vs Global vs Server State

### Local State Boundaries

Local state belongs to a single component or a small subtree. It should not leak into global stores.

Signs that state should remain local:
- Only one component reads it
- Only one component writes it
- It resets when the component unmounts (modal open state, form input value)
- It does not need to survive navigation

```typescript
// Local state: accordion panel expansion
function AccordionItem({ title, children }: AccordionItemProps) {
  const [isExpanded, setIsExpanded] = useState(false);

  return (
    <div>
      <button
        aria-expanded={isExpanded}
        onClick={() => setIsExpanded((prev) => !prev)}
      >
        {title}
      </button>
      {isExpanded && <div>{children}</div>}
    </div>
  );
}
```

### Global State Boundaries

Global state is needed when multiple distant components must share and react to the same data.

Signs that state should be global:
- Multiple unrelated components read it (theme, auth, locale)
- Changes must be reflected across the entire app instantly
- State persists across route changes

```typescript
// Zustand store for global UI preferences
interface UIStore {
  sidebarCollapsed: boolean;
  toggleSidebar: () => void;
  density: 'compact' | 'comfortable' | 'spacious';
  setDensity: (density: UIStore['density']) => void;
}

export const useUIStore = create<UIStore>()(
  persist(
    (set) => ({
      sidebarCollapsed: false,
      toggleSidebar: () => set((s) => ({ sidebarCollapsed: !s.sidebarCollapsed })),
      density: 'comfortable',
      setDensity: (density) => set({ density }),
    }),
    { name: 'ui-preferences' }
  )
);
```

### Server State Boundaries

Server state is any data that originates from an external source and has a lifecycle independent of the frontend application. It can become stale, needs refreshing, and may be modified by other clients.

Do not store server data in Redux or Zustand. Use TanStack Query or SWR which handle:
- Automatic background refetching
- Cache deduplication (multiple components requesting the same data share one network call)
- Stale-while-revalidate strategy
- Garbage collection of unused cache entries
- Retry with exponential backoff

```typescript
// Server state managed by TanStack Query
function useOrders(filters: OrderFilters) {
  return useQuery({
    queryKey: ['orders', filters],
    queryFn: () => api.fetchOrders(filters),
    staleTime: 30_000, // Data is fresh for 30 seconds
    refetchOnWindowFocus: true,
    refetchInterval: filters.status === 'pending' ? 10_000 : false,
  });
}
```

## State Machines with XState

For complex stateful workflows (multi-step forms, payment flows, drag-and-drop, authentication), use state machines to make impossible states impossible.

```typescript
import { createMachine, assign } from 'xstate';
import { useMachine } from '@xstate/react';

interface CheckoutContext {
  items: CartItem[];
  shippingAddress: Address | null;
  paymentMethod: PaymentMethod | null;
  orderId: string | null;
  error: string | null;
}

type CheckoutEvent =
  | { type: 'SET_SHIPPING'; address: Address }
  | { type: 'SET_PAYMENT'; method: PaymentMethod }
  | { type: 'SUBMIT' }
  | { type: 'BACK' }
  | { type: 'RETRY' };

const checkoutMachine = createMachine({
  id: 'checkout',
  initial: 'shipping',
  context: {
    items: [],
    shippingAddress: null,
    paymentMethod: null,
    orderId: null,
    error: null,
  } satisfies CheckoutContext,
  states: {
    shipping: {
      on: {
        SET_SHIPPING: {
          target: 'payment',
          actions: assign({ shippingAddress: ({ event }) => event.address }),
        },
      },
    },
    payment: {
      on: {
        SET_PAYMENT: {
          actions: assign({ paymentMethod: ({ event }) => event.method }),
        },
        SUBMIT: { target: 'processing' },
        BACK: { target: 'shipping' },
      },
    },
    processing: {
      invoke: {
        src: 'submitOrder',
        onDone: {
          target: 'confirmation',
          actions: assign({ orderId: ({ event }) => event.output.orderId }),
        },
        onError: {
          target: 'error',
          actions: assign({ error: ({ event }) => event.error.message }),
        },
      },
    },
    confirmation: { type: 'final' },
    error: {
      on: {
        RETRY: { target: 'processing' },
        BACK: { target: 'payment' },
      },
    },
  },
});

// Usage in component
function CheckoutWizard() {
  const [state, send] = useMachine(checkoutMachine);

  switch (true) {
    case state.matches('shipping'):
      return <ShippingForm onSubmit={(address) => send({ type: 'SET_SHIPPING', address })} />;
    case state.matches('payment'):
      return <PaymentForm onSubmit={() => send({ type: 'SUBMIT' })} onBack={() => send({ type: 'BACK' })} />;
    case state.matches('processing'):
      return <ProcessingSpinner />;
    case state.matches('confirmation'):
      return <OrderConfirmation orderId={state.context.orderId!} />;
    case state.matches('error'):
      return <ErrorScreen error={state.context.error!} onRetry={() => send({ type: 'RETRY' })} />;
  }
}
```

State machines prevent invalid transitions. The user cannot reach the payment step without setting a shipping address. The checkout cannot submit twice because the `processing` state has no `SUBMIT` transition.

## Optimistic Updates

To provide instant feedback while a server mutation is in progress, apply the change to local state immediately and roll back if the server responds with an error.

```typescript
function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (productId: string) => api.toggleFavorite(productId),

    onMutate: async (productId) => {
      // Cancel outgoing refetches to avoid overwriting optimistic update
      await queryClient.cancelQueries({ queryKey: ['products'] });

      // Snapshot the previous value
      const previousProducts = queryClient.getQueryData<Product[]>(['products']);

      // Optimistically update
      queryClient.setQueryData<Product[]>(['products'], (old) =>
        old?.map((p) =>
          p.id === productId ? { ...p, isFavorite: !p.isFavorite } : p
        ) ?? []
      );

      return { previousProducts };
    },

    onError: (_err, _productId, context) => {
      // Roll back on error
      if (context?.previousProducts) {
        queryClient.setQueryData(['products'], context.previousProducts);
      }
    },

    onSettled: () => {
      // Always refetch to ensure consistency
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

### Optimistic Update Guidelines

1. Always snapshot previous state in `onMutate`
2. Always roll back in `onError`
3. Always invalidate in `onSettled` to sync with server truth
4. Cancel in-flight queries before applying optimistic updates
5. Use optimistic updates only for interactions where latency matters (likes, toggles, reordering), not for destructive actions (delete, payment)

## Cache Invalidation

### Invalidation Strategies

```typescript
const queryClient = useQueryClient();

// Invalidate a specific query
queryClient.invalidateQueries({ queryKey: ['users', userId] });

// Invalidate all queries starting with 'users'
queryClient.invalidateQueries({ queryKey: ['users'] });

// Invalidate everything
queryClient.invalidateQueries();

// Remove from cache entirely (next access triggers fresh fetch)
queryClient.removeQueries({ queryKey: ['users', userId] });

// Set data directly (for mutations that return the updated entity)
queryClient.setQueryData(['users', userId], updatedUser);
```

### Cache Key Design

Design query keys hierarchically to enable precise invalidation.

```typescript
// Hierarchical keys
['products']                          // All products
['products', 'list', { category }]    // Filtered product list
['products', 'detail', productId]     // Single product
['products', 'detail', productId, 'reviews'] // Product reviews

// After updating a product:
queryClient.invalidateQueries({ queryKey: ['products'] }); // Invalidates all product queries
```

## Derived State

Never store state that can be computed from other state. Compute it inline or memoize with `useMemo`.

```typescript
// BAD: Storing derived state
const [items, setItems] = useState<Item[]>([]);
const [totalPrice, setTotalPrice] = useState(0); // Derived! Should not be stored.

useEffect(() => {
  setTotalPrice(items.reduce((sum, item) => sum + item.price * item.quantity, 0));
}, [items]);

// GOOD: Compute derived values
const [items, setItems] = useState<Item[]>([]);
const totalPrice = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

// GOOD: Memoize if computation is expensive
const totalPrice = useMemo(
  () => items.reduce((sum, item) => sum + item.price * item.quantity, 0),
  [items]
);
```

Derived state in Zustand:

```typescript
const useStore = create<Store>((set, get) => ({
  items: [],
  // Derived as a method, not stored state
  totalPrice: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
}));
```

Derived state in Jotai:

```typescript
const itemsAtom = atom<Item[]>([]);
const totalPriceAtom = atom((get) => {
  const items = get(itemsAtom);
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
});
```

## State Normalization

For complex relational data, normalize state to avoid deeply nested structures and data duplication.

```typescript
// BAD: Nested, duplicated data
interface DenormalizedState {
  posts: Array<{
    id: string;
    title: string;
    author: { id: string; name: string }; // Duplicated across posts
    comments: Array<{
      id: string;
      text: string;
      author: { id: string; name: string }; // Duplicated again
    }>;
  }>;
}

// GOOD: Normalized state
interface NormalizedState {
  users: Record<string, User>;
  posts: Record<string, Post>;
  comments: Record<string, Comment>;
}

interface Post {
  id: string;
  title: string;
  authorId: string; // Reference, not embedded
  commentIds: string[]; // References, not embedded
}
```

Normalization benefits:
- Single source of truth for each entity (update a user name in one place)
- O(1) lookups by ID
- Easier partial updates
- Smaller state diffs

TanStack Query handles normalization implicitly through its cache key design: each entity at its own key is effectively normalized. For client-side state, libraries like `normalizr` or manual normalization in reducers keep data flat.
