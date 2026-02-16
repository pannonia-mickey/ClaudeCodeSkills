---
name: Vue Testing
description: This skill should be used when the user asks about "Vue testing", "Vue Test Utils", "Vitest Vue", "Vue component testing", "Vue e2e testing", "Vue snapshot tests", "testing composables", "Pinia testing", or "Vue Playwright". It covers component testing with Vitest and Vue Test Utils, composable testing, store testing, and e2e patterns.
---

# Vue Testing

## Component Testing with Vitest + Vue Test Utils

```ts
import { mount } from '@vue/test-utils'
import { describe, it, expect, vi } from 'vitest'
import UserCard from '@/components/UserCard.vue'

describe('UserCard', () => {
  const defaultProps = {
    user: { id: '1', name: 'Alice', email: 'alice@test.com', isActive: true },
  }

  it('renders user information', () => {
    const wrapper = mount(UserCard, { props: defaultProps })

    expect(wrapper.text()).toContain('Alice')
    expect(wrapper.text()).toContain('alice@test.com')
  })

  it('emits delete event on button click', async () => {
    const wrapper = mount(UserCard, { props: defaultProps })

    await wrapper.find('[data-testid="delete-btn"]').trigger('click')

    expect(wrapper.emitted('delete')).toHaveLength(1)
    expect(wrapper.emitted('delete')![0]).toEqual(['1'])
  })

  it('shows active badge when user is active', () => {
    const wrapper = mount(UserCard, { props: defaultProps })
    expect(wrapper.find('.badge-active').exists()).toBe(true)
  })

  it('shows inactive state', () => {
    const wrapper = mount(UserCard, {
      props: { user: { ...defaultProps.user, isActive: false } },
    })
    expect(wrapper.find('.badge-active').exists()).toBe(false)
  })
})
```

## Testing with Slots

```ts
import { mount } from '@vue/test-utils'
import Modal from '@/components/Modal.vue'

describe('Modal', () => {
  it('renders slot content', () => {
    const wrapper = mount(Modal, {
      props: { show: true },
      slots: {
        default: '<p>Modal body</p>',
        header: '<h2>Title</h2>',
        footer: '<button>Close</button>',
      },
    })

    expect(wrapper.html()).toContain('Modal body')
    expect(wrapper.find('h2').text()).toBe('Title')
  })

  it('provides scoped slot data', () => {
    const wrapper = mount(DataList, {
      props: { items: [{ id: '1', name: 'Item 1' }] },
      slots: {
        default: `<template #default="{ item }">
          <span class="item">{{ item.name }}</span>
        </template>`,
      },
    })

    expect(wrapper.find('.item').text()).toBe('Item 1')
  })
})
```

## Testing Async Components

```ts
import { mount, flushPromises } from '@vue/test-utils'

describe('UserList', () => {
  it('loads and displays users', async () => {
    vi.spyOn(api, 'getUsers').mockResolvedValue([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ])

    const wrapper = mount(UserList)

    // Initially shows loading
    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(true)

    // Wait for async operations
    await flushPromises()

    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(false)
    expect(wrapper.findAll('[data-testid="user-item"]')).toHaveLength(2)
  })

  it('shows error state on API failure', async () => {
    vi.spyOn(api, 'getUsers').mockRejectedValue(new Error('Network error'))

    const wrapper = mount(UserList)
    await flushPromises()

    expect(wrapper.find('[data-testid="error"]').text()).toContain('Network error')
  })
})
```

## Testing with Router

```ts
import { mount } from '@vue/test-utils'
import { createRouter, createMemoryHistory } from 'vue-router'

function createTestRouter(initialRoute = '/') {
  return createRouter({
    history: createMemoryHistory(),
    routes: [
      { path: '/', component: { template: '<div>Home</div>' } },
      { path: '/users/:id', component: { template: '<div>User</div>' } },
    ],
  })
}

describe('Navigation', () => {
  it('navigates on link click', async () => {
    const router = createTestRouter()
    await router.push('/')

    const wrapper = mount(AppNav, {
      global: { plugins: [router] },
    })

    await wrapper.find('[data-testid="users-link"]').trigger('click')
    await router.isReady()

    expect(router.currentRoute.value.path).toBe('/users')
  })
})
```

## Testing with Pinia

```ts
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'

describe('UserDashboard', () => {
  it('displays users from store', () => {
    const wrapper = mount(UserDashboard, {
      global: {
        plugins: [
          createTestingPinia({
            initialState: {
              users: {
                users: [
                  { id: '1', name: 'Alice' },
                  { id: '2', name: 'Bob' },
                ],
              },
            },
          }),
        ],
      },
    })

    expect(wrapper.findAll('.user-row')).toHaveLength(2)
  })

  it('calls fetch action on mount', () => {
    const wrapper = mount(UserDashboard, {
      global: {
        plugins: [createTestingPinia({ stubActions: false })],
      },
    })

    const store = useUsersStore()
    expect(store.fetchUsers).toHaveBeenCalled()
  })
})
```

## References

- [Component Testing](references/component-testing.md) — Form testing, event testing, provide/inject mocking, suspense testing, snapshot testing.
- [E2E Patterns](references/e2e-patterns.md) — Playwright with Vue, page objects, test fixtures, visual regression, accessibility testing.
