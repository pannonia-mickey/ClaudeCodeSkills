# Component Testing

## Form Testing

```ts
import { mount } from '@vue/test-utils'

describe('LoginForm', () => {
  it('submits with valid credentials', async () => {
    const wrapper = mount(LoginForm)

    await wrapper.find('input[name="email"]').setValue('alice@test.com')
    await wrapper.find('input[name="password"]').setValue('password123')
    await wrapper.find('form').trigger('submit')

    expect(wrapper.emitted('submit')).toHaveLength(1)
    expect(wrapper.emitted('submit')![0]).toEqual([{
      email: 'alice@test.com',
      password: 'password123',
    }])
  })

  it('shows validation errors', async () => {
    const wrapper = mount(LoginForm)

    // Submit empty form
    await wrapper.find('form').trigger('submit')

    expect(wrapper.find('[data-testid="email-error"]').text()).toBe('Email is required')
    expect(wrapper.find('[data-testid="password-error"]').text()).toBe('Password is required')
    expect(wrapper.emitted('submit')).toBeUndefined()
  })

  it('disables submit while loading', async () => {
    const wrapper = mount(LoginForm, {
      props: { loading: true },
    })

    const submitBtn = wrapper.find('button[type="submit"]')
    expect(submitBtn.attributes('disabled')).toBeDefined()
  })
})
```

## Event Testing

```ts
describe('SearchInput', () => {
  it('debounces input events', async () => {
    vi.useFakeTimers()
    const wrapper = mount(SearchInput, {
      props: { debounce: 300 },
    })

    await wrapper.find('input').setValue('hel')
    await wrapper.find('input').setValue('hello')

    // Not emitted yet (debounced)
    expect(wrapper.emitted('search')).toBeUndefined()

    vi.advanceTimersByTime(300)
    await wrapper.vm.$nextTick()

    expect(wrapper.emitted('search')).toHaveLength(1)
    expect(wrapper.emitted('search')![0]).toEqual(['hello'])

    vi.useRealTimers()
  })

  it('handles keyboard events', async () => {
    const wrapper = mount(SearchInput)
    const input = wrapper.find('input')

    await input.setValue('test')
    await input.trigger('keydown.enter')

    expect(wrapper.emitted('submit')).toHaveLength(1)
  })
})
```

## Provide/Inject Mocking

```ts
import { mount } from '@vue/test-utils'

describe('ThemeToggle', () => {
  it('toggles theme', async () => {
    const toggleTheme = vi.fn()

    const wrapper = mount(ThemeToggle, {
      global: {
        provide: {
          [ThemeKey as symbol]: {
            theme: ref('light'),
            toggleTheme,
          },
        },
      },
    })

    await wrapper.find('button').trigger('click')
    expect(toggleTheme).toHaveBeenCalled()
  })
})

// Testing components that use inject with defaults
describe('UserAvatar', () => {
  it('renders without provider', () => {
    const wrapper = mount(UserAvatar, {
      props: { name: 'Alice' },
    })
    // Should use default inject values
    expect(wrapper.find('.avatar').exists()).toBe(true)
  })
})
```

## Testing Suspense

```ts
import { mount, flushPromises } from '@vue/test-utils'
import { Suspense } from 'vue'

describe('AsyncComponent', () => {
  it('renders after loading', async () => {
    const wrapper = mount({
      template: `
        <Suspense>
          <AsyncUserProfile :id="'123'" />
          <template #fallback>
            <div data-testid="loading">Loading...</div>
          </template>
        </Suspense>
      `,
      components: { AsyncUserProfile },
    })

    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(true)

    await flushPromises()

    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(false)
    expect(wrapper.find('[data-testid="user-name"]').exists()).toBe(true)
  })
})
```

## Snapshot Testing

```ts
import { mount } from '@vue/test-utils'

describe('Badge', () => {
  it.each(['success', 'warning', 'error', 'info'] as const)(
    'renders %s variant correctly',
    (variant) => {
      const wrapper = mount(Badge, {
        props: { variant, label: 'Test' },
      })
      expect(wrapper.html()).toMatchSnapshot()
    },
  )
})

// Inline snapshots for small components
it('renders correctly', () => {
  const wrapper = mount(Badge, {
    props: { variant: 'success', label: 'Active' },
  })
  expect(wrapper.html()).toMatchInlineSnapshot(`
    "<span class="badge badge-success">Active</span>"
  `)
})
```

## Testing v-model Components

```ts
describe('RatingInput', () => {
  it('updates v-model on click', async () => {
    const wrapper = mount(RatingInput, {
      props: { modelValue: 0, 'onUpdate:modelValue': (e: number) => wrapper.setProps({ modelValue: e }) },
    })

    await wrapper.findAll('.star')[2].trigger('click') // Click 3rd star

    expect(wrapper.props('modelValue')).toBe(3)
  })

  it('works with v-model', async () => {
    const Parent = defineComponent({
      components: { RatingInput },
      setup() {
        const rating = ref(0)
        return { rating }
      },
      template: '<RatingInput v-model="rating" />',
    })

    const wrapper = mount(Parent)
    await wrapper.findAll('.star')[4].trigger('click')

    expect(wrapper.vm.rating).toBe(5)
  })
})
```
