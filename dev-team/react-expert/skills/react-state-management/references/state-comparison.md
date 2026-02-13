# State Management Library Comparison

## Redux Toolkit

### When to Use

- Large application with many developers needing strict conventions
- Complex state interactions where actions trigger multiple state changes
- Need for time-travel debugging and state inspection via Redux DevTools
- Existing Redux codebase that needs modernization
- Middleware requirements: sagas, thunks, listeners for complex async flows

### Architecture

```typescript
// features/products/productsSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

interface ProductsState {
  items: Product[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  filters: ProductFilters;
}

const initialState: ProductsState = {
  items: [],
  status: 'idle',
  error: null,
  filters: { category: 'all', sortBy: 'name' },
};

export const fetchProducts = createAsyncThunk(
  'products/fetch',
  async (filters: ProductFilters) => {
    const response = await api.getProducts(filters);
    return response.data;
  }
);

const productsSlice = createSlice({
  name: 'products',
  initialState,
  reducers: {
    setFilters(state, action: PayloadAction<Partial<ProductFilters>>) {
      state.filters = { ...state.filters, ...action.payload };
    },
    clearFilters(state) {
      state.filters = initialState.filters;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message ?? 'Failed to fetch products';
      });
  },
});
```

### RTK Query

For server state within the Redux ecosystem, use RTK Query instead of writing thunks manually.

```typescript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const productsApi = createApi({
  reducerPath: 'productsApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Product'],
  endpoints: (builder) => ({
    getProducts: builder.query<Product[], ProductFilters>({
      query: (filters) => ({
        url: '/products',
        params: filters,
      }),
      providesTags: (result) =>
        result
          ? [
              ...result.map(({ id }) => ({ type: 'Product' as const, id })),
              { type: 'Product', id: 'LIST' },
            ]
          : [{ type: 'Product', id: 'LIST' }],
    }),
    updateProduct: builder.mutation<Product, Partial<Product> & Pick<Product, 'id'>>({
      query: ({ id, ...patch }) => ({
        url: `/products/${id}`,
        method: 'PATCH',
        body: patch,
      }),
      invalidatesTags: (_result, _error, { id }) => [{ type: 'Product', id }],
    }),
  }),
});

export const { useGetProductsQuery, useUpdateProductMutation } = productsApi;
```

### Performance

- Uses Immer for immutable updates with mutable syntax
- `createSelector` (Reselect) for memoized derived state
- Automatic structural sharing on selector results
- Normalized cache with `createEntityAdapter`
- Bundle: ~11 KB gzipped (RTK core)

## Zustand

### When to Use

- Small to medium applications needing simple global state
- Projects that value minimal boilerplate
- When state needs to be accessed outside React components (utilities, middleware)
- Quick prototyping where Redux overhead is not justified
- When you want React-independent state logic

### Architecture

```typescript
import { create } from 'zustand';
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface AuthStore {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
  updateProfile: (updates: Partial<User>) => void;
}

export const useAuthStore = create<AuthStore>()(
  devtools(
    persist(
      subscribeWithSelector(
        immer((set, get) => ({
          user: null,
          token: null,
          isAuthenticated: false,

          login: async (credentials) => {
            const response = await api.login(credentials);
            set((state) => {
              state.user = response.user;
              state.token = response.token;
              state.isAuthenticated = true;
            });
          },

          logout: () => {
            set((state) => {
              state.user = null;
              state.token = null;
              state.isAuthenticated = false;
            });
          },

          updateProfile: (updates) => {
            set((state) => {
              if (state.user) {
                Object.assign(state.user, updates);
              }
            });
          },
        }))
      ),
      {
        name: 'auth-storage',
        partialize: (state) => ({ token: state.token }), // Only persist token
      }
    ),
    { name: 'AuthStore' }
  )
);
```

### Selectors for Performance

Subscribe to specific slices to avoid unnecessary re-renders.

```typescript
// BAD: Re-renders on any store change
const store = useAuthStore();

// GOOD: Only re-renders when user changes
const user = useAuthStore((state) => state.user);
const isAuthenticated = useAuthStore((state) => state.isAuthenticated);

// GOOD: Derived selector with shallow comparison
import { shallow } from 'zustand/shallow';
const { user, isAuthenticated } = useAuthStore(
  (state) => ({ user: state.user, isAuthenticated: state.isAuthenticated }),
  shallow
);
```

### Outside React

Zustand stores work outside React components, making them ideal for utility functions and middleware.

```typescript
// In an API interceptor (no React component)
api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Subscribe to changes outside React
useAuthStore.subscribe(
  (state) => state.isAuthenticated,
  (isAuthenticated) => {
    if (!isAuthenticated) {
      router.push('/login');
    }
  }
);
```

### Performance

- No provider needed (no Context re-render issue)
- Selector-based subscriptions (component-level granularity)
- Bundle: ~1 KB gzipped
- Middleware composable: devtools, persist, immer, subscribeWithSelector

## Jotai

### When to Use

- Applications with highly granular, interconnected state
- When derived/computed state is a primary pattern
- When you want atom-level subscriptions (finest granularity)
- When state graph is complex with many derivations
- React Suspense integration for async atoms

### Architecture

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';
import { atomWithStorage } from 'jotai/utils';

// Primitive atoms
const searchQueryAtom = atom('');
const selectedCategoryAtom = atom<string | null>(null);
const sortOrderAtom = atomWithStorage<'asc' | 'desc'>('sortOrder', 'asc');

// Async atom (integrates with Suspense)
const productsAtom = atom(async () => {
  const response = await fetch('/api/products');
  return response.json() as Promise<Product[]>;
});

// Derived atom: combines multiple atoms
const filteredProductsAtom = atom((get) => {
  const products = get(productsAtom);
  const query = get(searchQueryAtom).toLowerCase();
  const category = get(selectedCategoryAtom);
  const sortOrder = get(sortOrderAtom);

  let filtered = products;

  if (query) {
    filtered = filtered.filter((p) => p.name.toLowerCase().includes(query));
  }

  if (category) {
    filtered = filtered.filter((p) => p.category === category);
  }

  return filtered.toSorted((a, b) =>
    sortOrder === 'asc'
      ? a.name.localeCompare(b.name)
      : b.name.localeCompare(a.name)
  );
});

// Read-write derived atom
const productCountAtom = atom(
  (get) => get(filteredProductsAtom).length,
  (_get, set, reset: boolean) => {
    if (reset) {
      set(searchQueryAtom, '');
      set(selectedCategoryAtom, null);
    }
  }
);

// Usage
function ProductSearch() {
  const [query, setQuery] = useAtom(searchQueryAtom);
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

function ProductCount() {
  const count = useAtomValue(productCountAtom); // Only re-renders when count changes
  return <span>{count} products</span>;
}
```

### Performance

- Atom-level subscriptions: components only re-render when their specific atoms change
- No selector functions needed (atoms are the selectors)
- Lazy evaluation: derived atoms only compute when read
- Bundle: ~2 KB gzipped
- Provider optional (default store or scoped stores)

## TanStack Query

### When to Use

- Any application that fetches data from APIs
- Real-time data that needs background refetching
- Paginated or infinite scroll data
- Optimistic updates on mutations
- Offline support with cache persistence

### Architecture

```typescript
import {
  useQuery,
  useMutation,
  useInfiniteQuery,
  useQueryClient,
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query';

// Query client configuration
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,        // Data fresh for 1 minute
      gcTime: 5 * 60_000,       // Garbage collect after 5 minutes
      retry: 3,                  // Retry failed requests 3 times
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
  },
});

// Paginated query
function useProducts(page: number, filters: ProductFilters) {
  return useQuery({
    queryKey: ['products', 'list', { page, ...filters }],
    queryFn: () => api.getProducts({ page, ...filters }),
    placeholderData: keepPreviousData, // Keep old data while fetching new page
  });
}

// Infinite query for infinite scroll
function useInfiniteProducts(filters: ProductFilters) {
  return useInfiniteQuery({
    queryKey: ['products', 'infinite', filters],
    queryFn: ({ pageParam }) => api.getProducts({ cursor: pageParam, ...filters }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });
}

// Prefetching for navigation
function ProductListItem({ product }: { product: Product }) {
  const queryClient = useQueryClient();

  const prefetchProduct = () => {
    queryClient.prefetchQuery({
      queryKey: ['products', 'detail', product.id],
      queryFn: () => api.getProduct(product.id),
      staleTime: 30_000,
    });
  };

  return (
    <Link
      to={`/products/${product.id}`}
      onMouseEnter={prefetchProduct}
      onFocus={prefetchProduct}
    >
      {product.name}
    </Link>
  );
}
```

### Performance

- Automatic request deduplication
- Background refetching with stale-while-revalidate
- Structural sharing on query results (prevents unnecessary re-renders)
- Garbage collection of unused cache entries
- Bundle: ~13 KB gzipped
- Window focus refetching, network reconnect refetching

## Migration Paths

### Redux to Zustand

1. Identify slices that can be migrated independently
2. Create Zustand stores matching slice shapes
3. Replace `useSelector` + `useDispatch` with Zustand hooks
4. Migrate middleware (thunks become async functions, sagas become subscriptions)
5. Remove the Redux provider once all slices are migrated

```typescript
// Redux slice
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },
    decrement: (state) => { state.value -= 1; },
  },
});

// Equivalent Zustand store
const useCounterStore = create<CounterStore>()(
  immer((set) => ({
    value: 0,
    increment: () => set((s) => { s.value += 1; }),
    decrement: () => set((s) => { s.value -= 1; }),
  }))
);
```

### Adding TanStack Query to Existing State

1. Install TanStack Query and set up the provider
2. Identify all API calls currently managed by Redux/Zustand
3. Migrate one endpoint at a time to `useQuery`/`useMutation`
4. Remove the corresponding state, actions, and reducers
5. The global store shrinks to only true client state

### Context API to Zustand

1. Identify contexts with performance issues (too many re-renders)
2. Create Zustand stores for frequently changing state
3. Keep Context for truly static or rarely changing values (theme, locale)
4. Replace `useContext` calls with Zustand selectors
5. Remove providers that are no longer needed

## Performance Comparison Summary

| Metric | Redux Toolkit | Zustand | Jotai | TanStack Query |
|---|---|---|---|---|
| Bundle Size | ~11 KB | ~1 KB | ~2 KB | ~13 KB |
| Re-render Granularity | Selector-based | Selector-based | Atom-level | Query key-based |
| Boilerplate | Medium | Low | Low | Medium |
| DevTools | Excellent | Good (via middleware) | Good (via devtools) | Excellent |
| SSR Support | Manual | Manual | Built-in | Built-in |
| Middleware | Rich ecosystem | Composable | Utilities package | Query plugins |
| Learning Curve | Medium-High | Low | Low-Medium | Medium |
| TypeScript DX | Good | Excellent | Excellent | Excellent |

The best approach for most new React applications: TanStack Query for server state + Zustand for global client state + useState/useReducer for local state. This combination covers all state categories with minimal overlap and excellent developer experience.
