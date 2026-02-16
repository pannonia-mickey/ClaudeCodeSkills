# Router Patterns

## Nested Layouts

```ts
// Layout-based route structure
const routes: RouteRecordRaw[] = [
  {
    path: '/',
    component: () => import('@/layouts/PublicLayout.vue'),
    children: [
      { path: '', component: () => import('@/pages/Home.vue') },
      { path: 'about', component: () => import('@/pages/About.vue') },
      { path: 'login', component: () => import('@/pages/Login.vue') },
    ],
  },
  {
    path: '/app',
    component: () => import('@/layouts/AppLayout.vue'),
    meta: { requiresAuth: true },
    children: [
      { path: '', redirect: '/app/dashboard' },
      { path: 'dashboard', component: () => import('@/pages/Dashboard.vue') },
      {
        path: 'settings',
        component: () => import('@/layouts/SettingsLayout.vue'),
        children: [
          { path: '', redirect: 'profile' },
          { path: 'profile', component: () => import('@/pages/settings/Profile.vue') },
          { path: 'security', component: () => import('@/pages/settings/Security.vue') },
          { path: 'billing', component: () => import('@/pages/settings/Billing.vue') },
        ],
      },
    ],
  },
]
```

```vue
<!-- layouts/AppLayout.vue -->
<template>
  <div class="app-layout">
    <AppSidebar />
    <main>
      <AppHeader />
      <RouterView v-slot="{ Component, route }">
        <Transition name="page" mode="out-in">
          <component :is="Component" :key="route.path" />
        </Transition>
      </RouterView>
    </main>
  </div>
</template>
```

## Route Transitions

```vue
<template>
  <RouterView v-slot="{ Component, route }">
    <Transition :name="route.meta.transition || 'fade'" mode="out-in">
      <KeepAlive :include="cachedViews">
        <component :is="Component" :key="route.path" />
      </KeepAlive>
    </Transition>
  </RouterView>
</template>

<style>
.fade-enter-active, .fade-leave-active { transition: opacity 0.2s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

.slide-enter-active, .slide-leave-active { transition: transform 0.3s ease; }
.slide-enter-from { transform: translateX(100%); }
.slide-leave-to { transform: translateX(-100%); }
</style>
```

## Dynamic Route Registration

```ts
// Plugin-based route registration
function registerModuleRoutes(router: Router, modules: Record<string, Module>) {
  for (const [name, module] of Object.entries(modules)) {
    if (module.routes) {
      module.routes.forEach(route => {
        router.addRoute('app', {
          ...route,
          path: `${name}/${route.path}`,
        })
      })
    }
  }
}

// Remove routes dynamically
const removeRoute = router.addRoute({ path: '/temp', component: TempPage })
// Later: removeRoute()
```

## Breadcrumbs

```ts
// Route meta for breadcrumbs
{
  path: 'users/:id',
  component: UserDetail,
  meta: {
    breadcrumb: (route: RouteLocationNormalized) => [
      { label: 'Home', to: '/' },
      { label: 'Users', to: '/users' },
      { label: `User ${route.params.id}`, to: route.path },
    ],
  },
}
```

```vue
<!-- Breadcrumb component -->
<script setup lang="ts">
const route = useRoute()

const breadcrumbs = computed(() => {
  const meta = route.meta.breadcrumb
  if (typeof meta === 'function') return meta(route)
  if (Array.isArray(meta)) return meta
  return []
})
</script>

<template>
  <nav aria-label="Breadcrumb">
    <ol>
      <li v-for="(crumb, i) in breadcrumbs" :key="crumb.to">
        <RouterLink v-if="i < breadcrumbs.length - 1" :to="crumb.to">
          {{ crumb.label }}
        </RouterLink>
        <span v-else aria-current="page">{{ crumb.label }}</span>
      </li>
    </ol>
  </nav>
</template>
```

## Query Parameters

```ts
// Typed query parameter composable
export function useQueryParam<T>(
  name: string,
  defaultValue: T,
  parse: (v: string) => T = (v) => v as unknown as T,
  serialize: (v: T) => string = String,
) {
  const router = useRouter()
  const route = useRoute()

  const value = computed<T>({
    get() {
      const raw = route.query[name]
      return raw ? parse(String(raw)) : defaultValue
    },
    set(newValue) {
      router.replace({
        query: {
          ...route.query,
          [name]: newValue === defaultValue ? undefined : serialize(newValue),
        },
      })
    },
  })

  return value
}

// Usage
const page = useQueryParam('page', 1, Number)
const search = useQueryParam('q', '')
const sort = useQueryParam<'asc' | 'desc'>('sort', 'asc')
```

## Route-Based Code Splitting

```ts
// Chunk naming for better debugging
const routes = [
  {
    path: '/dashboard',
    component: () => import(/* webpackChunkName: "dashboard" */ '@/pages/Dashboard.vue'),
  },
  {
    path: '/reports',
    component: () => import(/* webpackChunkName: "reports" */ '@/pages/Reports.vue'),
  },
]

// Prefetch on hover
const prefetchRoute = (routeName: string) => {
  const route = router.resolve({ name: routeName })
  if (route.matched[0]?.components?.default) {
    const component = route.matched[0].components.default
    if (typeof component === 'function') {
      (component as () => Promise<any>)()
    }
  }
}
```
