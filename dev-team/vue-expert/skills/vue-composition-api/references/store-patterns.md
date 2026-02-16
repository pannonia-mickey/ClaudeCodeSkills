# Store Patterns

## Store Composition

```ts
// Composing stores — one store using another
export const useCartStore = defineStore('cart', () => {
  const authStore = useAuthStore()
  const items = ref<CartItem[]>([])

  const total = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  async function checkout() {
    if (!authStore.isAuthenticated) {
      throw new Error('Must be logged in to checkout')
    }
    return api.createOrder({
      userId: authStore.user!.id,
      items: items.value,
    })
  }

  return { items, total, checkout }
})

// Shared composable used by multiple stores
function useLoadingState() {
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function withLoading<T>(fn: () => Promise<T>): Promise<T> {
    loading.value = true
    error.value = null
    try {
      return await fn()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'An error occurred'
      throw e
    } finally {
      loading.value = false
    }
  }

  return { loading, error, withLoading }
}

export const useProductsStore = defineStore('products', () => {
  const { loading, error, withLoading } = useLoadingState()
  const products = ref<Product[]>([])

  async function fetchProducts() {
    products.value = await withLoading(() => api.getProducts())
  }

  return { products, loading, error, fetchProducts }
})
```

## Optimistic Updates

```ts
export const useTodosStore = defineStore('todos', () => {
  const todos = ref<Todo[]>([])

  async function toggleTodo(id: string) {
    const todo = todos.value.find(t => t.id === id)
    if (!todo) return

    // Optimistic update
    const previousState = todo.completed
    todo.completed = !todo.completed

    try {
      await api.updateTodo(id, { completed: todo.completed })
    } catch {
      // Rollback on failure
      todo.completed = previousState
    }
  }

  async function deleteTodo(id: string) {
    const index = todos.value.findIndex(t => t.id === id)
    if (index === -1) return

    const removed = todos.value.splice(index, 1)[0]

    try {
      await api.deleteTodo(id)
    } catch {
      // Rollback
      todos.value.splice(index, 0, removed)
    }
  }

  return { todos, toggleTodo, deleteTodo }
})
```

## Store Subscriptions

```ts
// React to store changes
const usersStore = useUsersStore()

// Subscribe to state changes
usersStore.$subscribe((mutation, state) => {
  console.log('Mutation type:', mutation.type)
  console.log('New state:', state)

  // Persist to localStorage
  localStorage.setItem('users-state', JSON.stringify(state))
})

// Subscribe to actions
usersStore.$onAction(({ name, args, after, onError }) => {
  const startTime = Date.now()
  console.log(`Action ${name} called with:`, args)

  after((result) => {
    console.log(`Action ${name} completed in ${Date.now() - startTime}ms`)
  })

  onError((error) => {
    console.error(`Action ${name} failed:`, error)
  })
})
```

## Testing Stores

```ts
import { setActivePinia, createPinia } from 'pinia'
import { describe, it, expect, beforeEach, vi } from 'vitest'

describe('useUsersStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('fetches users', async () => {
    const store = useUsersStore()
    expect(store.users).toEqual([])
    expect(store.loading).toBe(false)

    // Mock API
    vi.spyOn(api, 'getUsers').mockResolvedValue([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ])

    await store.fetchUsers()

    expect(store.users).toHaveLength(2)
    expect(store.loading).toBe(false)
  })

  it('handles fetch error', async () => {
    const store = useUsersStore()
    vi.spyOn(api, 'getUsers').mockRejectedValue(new Error('Network error'))

    await store.fetchUsers()

    expect(store.error).toBe('Network error')
    expect(store.users).toEqual([])
  })

  it('computes active users', async () => {
    const store = useUsersStore()
    store.users = [
      { id: '1', name: 'Alice', isActive: true },
      { id: '2', name: 'Bob', isActive: false },
      { id: '3', name: 'Carol', isActive: true },
    ]

    expect(store.activeUsers).toHaveLength(2)
  })
})
```

## Store Plugins

```ts
// Custom Pinia plugin
function piniaLogger({ store }: PiniaPluginContext) {
  store.$subscribe((mutation) => {
    console.log(`[${store.$id}] ${mutation.type}`)
  })

  store.$onAction(({ name, after }) => {
    console.log(`[${store.$id}] action: ${name}`)
    after(() => console.log(`[${store.$id}] action: ${name} completed`))
  })
}

// Error tracking plugin
function piniaErrorTracker({ store }: PiniaPluginContext) {
  store.$onAction(({ name, onError }) => {
    onError((error) => {
      errorTracker.captureException(error, {
        tags: { store: store.$id, action: name },
      })
    })
  })
}

const pinia = createPinia()
pinia.use(piniaLogger)
pinia.use(piniaErrorTracker)
```
