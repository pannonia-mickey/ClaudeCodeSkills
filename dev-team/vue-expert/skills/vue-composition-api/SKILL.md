---
name: Vue Composition API
description: This skill should be used when the user asks about "Vue Router", "Pinia", "Vue state management", "Vue router guards", "Vue route params", "defineStore", "Vue navigation", "Vue route middleware", or "Pinia persist". It covers Vue Router 4, Pinia stores, navigation guards, and state management patterns.
---

# Vue Composition API — Router & State

## Vue Router 4

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouteRecordRaw } from 'vue-router'

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    component: () => import('@/layouts/DefaultLayout.vue'),
    children: [
      { path: '', name: 'home', component: () => import('@/pages/Home.vue') },
      { path: 'users', name: 'users', component: () => import('@/pages/Users.vue') },
      {
        path: 'users/:id',
        name: 'user-detail',
        component: () => import('@/pages/UserDetail.vue'),
        props: true,
      },
    ],
  },
  {
    path: '/admin',
    component: () => import('@/layouts/AdminLayout.vue'),
    meta: { requiresAuth: true, roles: ['admin'] },
    children: [
      { path: '', name: 'admin-dashboard', component: () => import('@/pages/admin/Dashboard.vue') },
    ],
  },
  { path: '/:pathMatch(.*)*', name: 'not-found', component: () => import('@/pages/NotFound.vue') },
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    return savedPosition || { top: 0 }
  },
})

export default router
```

## Navigation Guards

```ts
// Global guard — auth check
router.beforeEach(async (to, from) => {
  const auth = useAuthStore()

  if (to.meta.requiresAuth && !auth.isAuthenticated) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }

  if (to.meta.roles && !to.meta.roles.includes(auth.user?.role)) {
    return { name: 'forbidden' }
  }
})

// Per-route guard
{
  path: 'users/:id/edit',
  component: () => import('@/pages/UserEdit.vue'),
  beforeEnter: async (to) => {
    const user = await fetchUser(to.params.id as string)
    if (!user) return { name: 'not-found' }
  },
}

// In-component guard with Composition API
import { onBeforeRouteLeave } from 'vue-router'

onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) {
    return confirm('You have unsaved changes. Leave anyway?')
  }
})

// Route params in composition API
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

const userId = computed(() => route.params.id as string)

function navigateToUser(id: string) {
  router.push({ name: 'user-detail', params: { id } })
}
```

## Pinia State Management

```ts
// stores/users.ts
import { defineStore } from 'pinia'

interface UsersState {
  users: User[]
  currentUser: User | null
  loading: boolean
  error: string | null
}

export const useUsersStore = defineStore('users', () => {
  // State
  const users = ref<User[]>([])
  const currentUser = ref<User | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  // Getters
  const activeUsers = computed(() => users.value.filter(u => u.isActive))
  const userById = computed(() => (id: string) => users.value.find(u => u.id === id))

  // Actions
  async function fetchUsers() {
    loading.value = true
    error.value = null
    try {
      users.value = await api.getUsers()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Failed to fetch users'
    } finally {
      loading.value = false
    }
  }

  async function createUser(data: CreateUserInput) {
    const user = await api.createUser(data)
    users.value.push(user)
    return user
  }

  async function deleteUser(id: string) {
    await api.deleteUser(id)
    users.value = users.value.filter(u => u.id !== id)
  }

  function $reset() {
    users.value = []
    currentUser.value = null
    loading.value = false
    error.value = null
  }

  return { users, currentUser, loading, error, activeUsers, userById, fetchUsers, createUser, deleteUser, $reset }
})

// Usage in component
const store = useUsersStore()
await store.fetchUsers()
const user = store.userById('123')
```

## Pinia Persistence

```ts
// With pinia-plugin-persistedstate
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)

export const useAuthStore = defineStore('auth', () => {
  const token = ref<string | null>(null)
  const user = ref<User | null>(null)

  const isAuthenticated = computed(() => !!token.value)

  return { token, user, isAuthenticated }
}, {
  persist: {
    pick: ['token'],    // Only persist token, not full user
    storage: localStorage,
  },
})
```

## References

- [Router Patterns](references/router-patterns.md) — Nested layouts, route transitions, dynamic routes, breadcrumbs, query parameters, route-based code splitting.
- [Store Patterns](references/store-patterns.md) — Store composition, optimistic updates, subscriptions, testing stores, store plugins, devtools integration.
