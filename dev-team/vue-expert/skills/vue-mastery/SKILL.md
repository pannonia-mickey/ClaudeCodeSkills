---
name: Vue Mastery
description: This skill should be used when the user asks about "Vue 3", "Composition API", "Vue reactivity", "Vue components", "script setup", "Vue ref", "Vue computed", "Vue watch", "Vue lifecycle", "Vue slots", or "Vue provide inject". It covers Composition API, reactivity system, component patterns, slots, and lifecycle hooks.
---

# Vue Mastery

## Script Setup with TypeScript

```vue
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'
import type { User } from '@/types'

// Props with TypeScript
const props = defineProps<{
  userId: string
  initialData?: User
}>()

// Props with defaults
const props = withDefaults(defineProps<{
  title: string
  variant?: 'primary' | 'secondary'
  disabled?: boolean
}>(), {
  variant: 'primary',
  disabled: false,
})

// Events with TypeScript
const emit = defineEmits<{
  update: [value: string]
  delete: [id: string]
  submit: [data: User]
}>()

// Expose public methods
defineExpose({
  reset,
  validate,
})

// Reactive state
const count = ref(0)
const user = ref<User | null>(null)
const loading = ref(false)

// Computed
const fullName = computed(() =>
  user.value ? `${user.value.firstName} ${user.value.lastName}` : ''
)

// Watcher
watch(() => props.userId, async (newId) => {
  loading.value = true
  user.value = await fetchUser(newId)
  loading.value = false
}, { immediate: true })

// Lifecycle
onMounted(() => {
  console.log('Component mounted')
})
</script>

<template>
  <div>
    <h2>{{ fullName }}</h2>
    <p>Count: {{ count }}</p>
    <button @click="count++">Increment</button>
  </div>
</template>
```

## Reactivity System

```vue
<script setup lang="ts">
import { ref, reactive, computed, watch, watchEffect, toRefs, shallowRef } from 'vue'

// ref — for primitives and single values
const count = ref(0)           // Access: count.value
const name = ref('Alice')     // Auto-unwrapped in template

// reactive — for objects (deeply reactive)
const state = reactive({
  users: [] as User[],
  filter: '',
  page: 1,
})
// Access: state.users (no .value needed)

// GOTCHA: Don't destructure reactive — loses reactivity
// BAD: const { users, filter } = state
// GOOD: use toRefs
const { users, filter } = toRefs(state)

// computed — cached derived state
const filteredUsers = computed(() =>
  state.users.filter(u => u.name.includes(state.filter))
)

// watch — react to specific changes
watch(count, (newVal, oldVal) => {
  console.log(`Count changed: ${oldVal} → ${newVal}`)
})

// Watch multiple sources
watch([count, name], ([newCount, newName]) => {
  console.log(`Count: ${newCount}, Name: ${newName}`)
})

// watchEffect — auto-tracks dependencies
watchEffect(() => {
  // Runs whenever count.value or name.value changes
  document.title = `${name.value} (${count.value})`
})

// shallowRef — only track .value reassignment, not deep changes
const largeList = shallowRef<Item[]>([])
// Must replace the whole array to trigger reactivity
largeList.value = [...largeList.value, newItem]
</script>
```

## Component Patterns

```vue
<!-- Generic list component with slots -->
<script setup lang="ts" generic="T extends { id: string }">
defineProps<{
  items: T[]
  loading?: boolean
}>()

defineSlots<{
  default: (props: { item: T; index: number }) => any
  empty: () => any
  loading: () => any
}>()
</script>

<template>
  <div class="list">
    <div v-if="loading">
      <slot name="loading">
        <p>Loading...</p>
      </slot>
    </div>
    <div v-else-if="items.length === 0">
      <slot name="empty">
        <p>No items found</p>
      </slot>
    </div>
    <div v-else v-for="(item, index) in items" :key="item.id">
      <slot :item="item" :index="index" />
    </div>
  </div>
</template>
```

```vue
<!-- Usage -->
<GenericList :items="users" :loading="isLoading">
  <template #default="{ item: user }">
    <UserCard :user="user" @click="selectUser(user)" />
  </template>
  <template #empty>
    <EmptyState message="No users found" />
  </template>
</GenericList>
```

## Provide / Inject

```ts
// Typed provide/inject
import { provide, inject } from 'vue'
import type { InjectionKey } from 'vue'

interface ThemeContext {
  theme: Ref<'light' | 'dark'>
  toggleTheme: () => void
}

const ThemeKey: InjectionKey<ThemeContext> = Symbol('theme')

// Provider component
const theme = ref<'light' | 'dark'>('light')
provide(ThemeKey, {
  theme,
  toggleTheme: () => {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
  },
})

// Consumer component
function useTheme() {
  const ctx = inject(ThemeKey)
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider')
  return ctx
}
```

## References

- [Composables](references/composables.md) — Composable patterns, useAsync, useFetch, useLocalStorage, useDebounce, composable testing.
- [Advanced Components](references/advanced-components.md) — Render functions, functional components, dynamic components, teleport, transitions, v-model patterns.
