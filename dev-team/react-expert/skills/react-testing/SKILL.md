---
name: React Testing
description: This skill should be used when the user asks about "React Testing Library", "test React component", "Jest React", "Vitest React", "test hook", or "MSW". It covers React Testing Library setup, component and hook testing, async patterns, mocking strategies with MSW, and the testing philosophy of testing behavior over implementation.
---

### Testing Philosophy

Write tests that resemble how users interact with the application. Do not test implementation details like internal state, component instance methods, or lifecycle calls. Test what the user sees and does.

The core principles:
1. Render the component
2. Interact with it as a user would (click, type, select)
3. Assert on what the user would see (text, roles, visibility)

```typescript
// BAD: Testing implementation details
expect(component.state.isOpen).toBe(true);
expect(wrapper.find('InternalDropdown').prop('visible')).toBe(true);

// GOOD: Testing user-visible behavior
await userEvent.click(screen.getByRole('button', { name: 'Open menu' }));
expect(screen.getByRole('menu')).toBeVisible();
```

### React Testing Library Setup

Configure a custom render function that wraps components with necessary providers. This avoids repeating provider setup in every test file.

```typescript
// test/setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';

afterEach(() => {
  cleanup();
});

// test/render.tsx
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ThemeProvider } from '@/providers/ThemeProvider';
import { MemoryRouter } from 'react-router-dom';

interface CustomRenderOptions extends Omit<RenderOptions, 'wrapper'> {
  initialRoute?: string;
  queryClient?: QueryClient;
}

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  });
}

export function renderWithProviders(
  ui: React.ReactElement,
  {
    initialRoute = '/',
    queryClient = createTestQueryClient(),
    ...options
  }: CustomRenderOptions = {}
) {
  function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        <ThemeProvider>
          <MemoryRouter initialEntries={[initialRoute]}>
            {children}
          </MemoryRouter>
        </ThemeProvider>
      </QueryClientProvider>
    );
  }

  return {
    ...render(ui, { wrapper: Wrapper, ...options }),
    queryClient,
  };
}
```

### Component Testing

To test a component, render it with realistic props and assert on the resulting DOM using accessible queries.

```typescript
import { screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { renderWithProviders } from '@/test/render';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('submits credentials and shows success', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn().mockResolvedValue({ success: true });
    renderWithProviders(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'user@example.com');
    await user.type(screen.getByLabelText('Password'), 'securepassword');
    await user.click(screen.getByRole('button', { name: 'Sign in' }));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'securepassword',
    });
    expect(await screen.findByText('Welcome back!')).toBeInTheDocument();
  });

  it('displays validation errors for empty fields', async () => {
    const user = userEvent.setup();
    renderWithProviders(<LoginForm onSubmit={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: 'Sign in' }));

    expect(screen.getByText('Email is required')).toBeInTheDocument();
    expect(screen.getByText('Password is required')).toBeInTheDocument();
  });

  it('disables submit button while loading', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn().mockImplementation(
      () => new Promise((resolve) => setTimeout(resolve, 1000))
    );
    renderWithProviders(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'user@example.com');
    await user.type(screen.getByLabelText('Password'), 'password');
    await user.click(screen.getByRole('button', { name: 'Sign in' }));

    expect(screen.getByRole('button', { name: 'Signing in...' })).toBeDisabled();
  });
});
```

### Hook Testing with renderHook

To test custom hooks in isolation, use `renderHook` from React Testing Library.

```typescript
import { renderHook, act, waitFor } from '@testing-library/react';
import { useDebounce } from './useDebounce';
import { useCounter } from './useCounter';

describe('useDebounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('debounces the value by the specified delay', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'hello', delay: 300 } }
    );

    expect(result.current).toBe('hello');

    rerender({ value: 'hello world', delay: 300 });
    expect(result.current).toBe('hello'); // Still old value

    act(() => {
      vi.advanceTimersByTime(300);
    });
    expect(result.current).toBe('hello world'); // Updated after delay
  });
});

describe('useCounter', () => {
  it('increments and decrements', () => {
    const { result } = renderHook(() => useCounter(0));

    act(() => {
      result.current.increment();
    });
    expect(result.current.count).toBe(1);

    act(() => {
      result.current.decrement();
    });
    expect(result.current.count).toBe(0);
  });
});
```

Wrap all state-updating calls in `act()`. When testing hooks that depend on providers, pass a wrapper to `renderHook`.

### Async Testing

To test components that fetch data or perform async operations, use `findBy` queries (which wait for elements to appear) and `waitFor` for assertions that need to poll.

```typescript
import { screen, waitFor } from '@testing-library/react';
import { renderWithProviders } from '@/test/render';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  it('loads and displays user data', async () => {
    renderWithProviders(<UserProfile userId="123" />);

    // Shows loading state first
    expect(screen.getByRole('progressbar')).toBeInTheDocument();

    // Waits for data to load (findBy = getBy + waitFor)
    const heading = await screen.findByRole('heading', { name: 'Jane Doe' });
    expect(heading).toBeInTheDocument();
    expect(screen.getByText('jane@example.com')).toBeInTheDocument();

    // Loading indicator should be gone
    expect(screen.queryByRole('progressbar')).not.toBeInTheDocument();
  });

  it('displays error state when fetch fails', async () => {
    // MSW handler returns error for this test
    server.use(
      http.get('/api/users/123', () => {
        return HttpResponse.json(
          { message: 'User not found' },
          { status: 404 }
        );
      })
    );

    renderWithProviders(<UserProfile userId="123" />);

    expect(await screen.findByRole('alert')).toHaveTextContent('User not found');
  });
});
```

Never use arbitrary `setTimeout` or `waitFor` with fixed timeouts. Rely on `findBy` queries and let the testing framework handle timing.

### Mocking with MSW

To mock API calls, use Mock Service Worker (MSW) to intercept requests at the network level. This tests the full data-fetching flow without changing component code.

```typescript
// test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: 'Jane Doe',
      email: 'jane@example.com',
    });
  }),

  http.post('/api/login', async ({ request }) => {
    const body = await request.json();
    if (body.email === 'user@example.com') {
      return HttpResponse.json({ token: 'mock-jwt-token' });
    }
    return HttpResponse.json(
      { message: 'Invalid credentials' },
      { status: 401 }
    );
  }),
];

// test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// test/setup.ts
import { server } from './mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Set `onUnhandledRequest: 'error'` to catch any API call that does not have a handler, preventing silent test failures.

Override handlers per-test for error scenarios or edge cases using `server.use()`.

### Testing Patterns Summary

| What to Test | How to Test It |
|---|---|
| Component renders correctly | Render + assert on accessible elements |
| User interactions | `userEvent.click`, `userEvent.type` + assert on changes |
| Async data loading | MSW handlers + `findBy` queries |
| Error states | MSW error handlers + assert on error UI |
| Custom hooks | `renderHook` + `act` for state updates |
| Form validation | Type invalid input + submit + assert on error messages |
| Accessibility | `toBeVisible`, `toHaveAccessibleName`, role queries |

## References

- [testing-patterns.md](references/testing-patterns.md) - Detailed testing patterns including user-event testing, async testing, snapshot testing, integration tests, MSW patterns, testing custom hooks, context providers, and error boundaries.
- [testing-tools.md](references/testing-tools.md) - RTL query priority guide, custom render utilities, test coverage strategies, Vitest vs Jest comparison, and @testing-library/user-event v14 API.
