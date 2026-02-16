# Composables

## Composable Conventions

```ts
// Naming: use* prefix
// Return: reactive state + methods
// Accept: refs or plain values (use toValue for flexibility)

import { ref, toValue, watchEffect, type MaybeRefOrGetter } from 'vue'

// useAsync — generic async data fetching
export function useAsync<T>(
  fn: () => Promise<T>,
  options?: { immediate?: boolean }
) {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      data.value = await fn()
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      loading.value = false
    }
  }

  if (options?.immediate !== false) {
    execute()
  }

  return { data, error, loading, execute }
}

// Usage
const { data: users, loading, error, execute: refresh } = useAsync(
  () => api.getUsers()
)
```

## useFetch

```ts
import { ref, toValue, watchEffect, type MaybeRefOrGetter } from 'vue'

export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  watchEffect(async () => {
    const resolvedUrl = toValue(url)
    if (!resolvedUrl) return

    loading.value = true
    error.value = null

    try {
      const response = await fetch(resolvedUrl)
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      data.value = await response.json()
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      loading.value = false
    }
  })

  return { data, error, loading }
}

// Reactive URL — refetches when userId changes
const userId = ref('123')
const { data: user, loading } = useFetch<User>(
  () => `/api/users/${userId.value}`
)
```

## useLocalStorage

```ts
import { ref, watch } from 'vue'

export function useLocalStorage<T>(key: string, defaultValue: T) {
  const stored = localStorage.getItem(key)
  const data = ref<T>(stored ? JSON.parse(stored) : defaultValue) as Ref<T>

  watch(data, (newValue) => {
    if (newValue === null || newValue === undefined) {
      localStorage.removeItem(key)
    } else {
      localStorage.setItem(key, JSON.stringify(newValue))
    }
  }, { deep: true })

  return data
}

// Usage
const theme = useLocalStorage<'light' | 'dark'>('theme', 'light')
const favorites = useLocalStorage<string[]>('favorites', [])
```

## useDebounce

```ts
import { ref, watch, type Ref } from 'vue'

export function useDebounce<T>(source: Ref<T>, delay = 300): Ref<T> {
  const debounced = ref(source.value) as Ref<T>
  let timeout: ReturnType<typeof setTimeout>

  watch(source, (newValue) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      debounced.value = newValue
    }, delay)
  })

  return debounced
}

// Usage in search
const searchQuery = ref('')
const debouncedQuery = useDebounce(searchQuery, 500)

// Only fetches after 500ms of inactivity
watch(debouncedQuery, (query) => {
  fetchResults(query)
})
```

## usePagination

```ts
export function usePagination<T>(
  fetchFn: (page: number, pageSize: number) => Promise<{ items: T[]; total: number }>,
  options = { pageSize: 20 }
) {
  const items = ref<T[]>([]) as Ref<T[]>
  const page = ref(1)
  const total = ref(0)
  const loading = ref(false)
  const pageSize = ref(options.pageSize)

  const totalPages = computed(() => Math.ceil(total.value / pageSize.value))
  const hasNext = computed(() => page.value < totalPages.value)
  const hasPrev = computed(() => page.value > 1)

  async function fetchPage(p: number) {
    loading.value = true
    try {
      const result = await fetchFn(p, pageSize.value)
      items.value = result.items
      total.value = result.total
      page.value = p
    } finally {
      loading.value = false
    }
  }

  function next() { if (hasNext.value) fetchPage(page.value + 1) }
  function prev() { if (hasPrev.value) fetchPage(page.value - 1) }
  function goTo(p: number) { fetchPage(Math.max(1, Math.min(p, totalPages.value))) }

  fetchPage(1)

  return { items, page, total, totalPages, hasNext, hasPrev, loading, next, prev, goTo }
}
```

## Composable Testing

```ts
// Test composables by mounting them in a component context
import { mount } from '@vue/test-utils'
import { defineComponent } from 'vue'

function withSetup<T>(composable: () => T) {
  let result: T
  const wrapper = mount(defineComponent({
    setup() {
      result = composable()
      return {}
    },
    template: '<div />',
  }))
  return { result: result!, wrapper }
}

describe('useCounter', () => {
  it('increments count', () => {
    const { result } = withSetup(() => useCounter())
    expect(result.count.value).toBe(0)

    result.increment()
    expect(result.count.value).toBe(1)
  })
})

// Or use @vue/test-utils renderComposable (if available)
import { renderComposable } from '@/test-utils'

test('useDebounce', async () => {
  vi.useFakeTimers()
  const source = ref('initial')
  const debounced = useDebounce(source, 300)

  source.value = 'updated'
  expect(debounced.value).toBe('initial')

  vi.advanceTimersByTime(300)
  expect(debounced.value).toBe('updated')
})
```
