# Advanced Components

## v-model Patterns

```vue
<!-- Single v-model -->
<script setup lang="ts">
const model = defineModel<string>()
// Equivalent to: props.modelValue + emit('update:modelValue')
</script>

<template>
  <input :value="model" @input="model = ($event.target as HTMLInputElement).value" />
</template>

<!-- Named v-model -->
<script setup lang="ts">
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')
</script>

<!-- Usage: <UserForm v-model:first-name="first" v-model:last-name="last" /> -->

<!-- v-model with transform -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string, 'capitalize' | 'trim'>()

watch(model, (val) => {
  if (!val) return
  let transformed = val
  if (modifiers.trim) transformed = transformed.trim()
  if (modifiers.capitalize) transformed = transformed.charAt(0).toUpperCase() + transformed.slice(1)
  if (transformed !== val) model.value = transformed
})
</script>

<!-- Usage: <TextInput v-model.capitalize.trim="name" /> -->
```

## Dynamic Components

```vue
<script setup lang="ts">
import { shallowRef, defineAsyncComponent } from 'vue'

const tabs = {
  profile: defineAsyncComponent(() => import('./ProfileTab.vue')),
  settings: defineAsyncComponent(() => import('./SettingsTab.vue')),
  billing: defineAsyncComponent({
    loader: () => import('./BillingTab.vue'),
    loadingComponent: LoadingSpinner,
    errorComponent: ErrorDisplay,
    delay: 200,
    timeout: 10000,
  }),
} as const

type TabKey = keyof typeof tabs
const activeTab = shallowRef<TabKey>('profile')
</script>

<template>
  <nav>
    <button
      v-for="(_, key) in tabs"
      :key="key"
      :class="{ active: activeTab === key }"
      @click="activeTab = key"
    >
      {{ key }}
    </button>
  </nav>
  <KeepAlive :max="5">
    <component :is="tabs[activeTab]" />
  </KeepAlive>
</template>
```

## Render Functions

```ts
import { h, defineComponent, type PropType } from 'vue'

// When templates are too limiting
export const VirtualList = defineComponent({
  props: {
    items: { type: Array as PropType<any[]>, required: true },
    itemHeight: { type: Number, default: 40 },
    containerHeight: { type: Number, default: 400 },
  },
  setup(props, { slots }) {
    const scrollTop = ref(0)

    const visibleItems = computed(() => {
      const start = Math.floor(scrollTop.value / props.itemHeight)
      const count = Math.ceil(props.containerHeight / props.itemHeight) + 1
      return props.items.slice(start, start + count).map((item, i) => ({
        item,
        index: start + i,
      }))
    })

    return () => h('div', {
      style: { height: `${props.containerHeight}px`, overflow: 'auto' },
      onScroll: (e: Event) => {
        scrollTop.value = (e.target as HTMLElement).scrollTop
      },
    }, [
      h('div', { style: { height: `${props.items.length * props.itemHeight}px`, position: 'relative' } },
        visibleItems.value.map(({ item, index }) =>
          h('div', {
            key: index,
            style: {
              position: 'absolute',
              top: `${index * props.itemHeight}px`,
              height: `${props.itemHeight}px`,
              width: '100%',
            },
          }, slots.default?.({ item, index }))
        )
      ),
    ])
  },
})
```

## Teleport

```vue
<!-- Modal teleported to body -->
<script setup lang="ts">
const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <Teleport to="body">
    <Transition name="modal">
      <div v-if="showModal" class="modal-overlay" @click.self="showModal = false">
        <div class="modal-content">
          <slot />
          <button @click="showModal = false">Close</button>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<!-- Conditional teleport target -->
<Teleport :to="isMobile ? '#mobile-sidebar' : '#desktop-sidebar'" :disabled="!shouldTeleport">
  <NavigationMenu />
</Teleport>
```

## Transitions

```vue
<script setup lang="ts">
import { ref } from 'vue'

const items = ref([
  { id: 1, text: 'Item 1' },
  { id: 2, text: 'Item 2' },
])

function addItem() {
  items.value.push({ id: Date.now(), text: `Item ${items.value.length + 1}` })
}

function removeItem(id: number) {
  items.value = items.value.filter(i => i.id !== id)
}
</script>

<template>
  <button @click="addItem">Add</button>

  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id" @click="removeItem(item.id)">
      {{ item.text }}
    </li>
  </TransitionGroup>
</template>

<style>
.list-enter-active, .list-leave-active {
  transition: all 0.3s ease;
}
.list-enter-from, .list-leave-to {
  opacity: 0;
  transform: translateX(-30px);
}
.list-move {
  transition: transform 0.3s ease;
}
</style>
```

## Functional Components

```ts
// Lightweight wrapper components
import type { FunctionalComponent } from 'vue'

interface IconProps {
  name: string
  size?: number
  color?: string
}

const Icon: FunctionalComponent<IconProps> = (props, { attrs }) => {
  return h('svg', {
    width: props.size ?? 24,
    height: props.size ?? 24,
    fill: props.color ?? 'currentColor',
    ...attrs,
  }, [
    h('use', { href: `#icon-${props.name}` }),
  ])
}

Icon.props = ['name', 'size', 'color']
Icon.displayName = 'Icon'

export default Icon
```
