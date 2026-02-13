# React Testing Patterns

## User-Event Testing

The `@testing-library/user-event` library simulates real user interactions more accurately than `fireEvent`. Always use `userEvent.setup()` to create a user instance before interactions.

```typescript
import userEvent from '@testing-library/user-event';

describe('Dropdown', () => {
  it('opens on click and selects an option', async () => {
    const user = userEvent.setup();
    const onSelect = vi.fn();
    render(<Dropdown options={['Apple', 'Banana', 'Cherry']} onSelect={onSelect} />);

    await user.click(screen.getByRole('combobox'));
    expect(screen.getByRole('listbox')).toBeVisible();

    await user.click(screen.getByRole('option', { name: 'Banana' }));
    expect(onSelect).toHaveBeenCalledWith('Banana');
    expect(screen.queryByRole('listbox')).not.toBeInTheDocument();
  });

  it('supports keyboard navigation', async () => {
    const user = userEvent.setup();
    render(<Dropdown options={['Apple', 'Banana', 'Cherry']} onSelect={vi.fn()} />);

    await user.tab(); // Focus the combobox
    await user.keyboard('{Enter}'); // Open
    await user.keyboard('{ArrowDown}'); // Move to first option
    await user.keyboard('{ArrowDown}'); // Move to second option
    await user.keyboard('{Enter}'); // Select

    expect(screen.getByRole('combobox')).toHaveTextContent('Banana');
  });
});
```

### Text Input

```typescript
it('filters results as user types', async () => {
  const user = userEvent.setup();
  render(<SearchableList items={mockItems} />);

  const input = screen.getByRole('searchbox');
  await user.type(input, 'react');

  // Assert filtered results
  expect(screen.getByText('React Hooks')).toBeInTheDocument();
  expect(screen.queryByText('Vue Composition API')).not.toBeInTheDocument();

  // Clear and retype
  await user.clear(input);
  await user.type(input, 'vue');

  expect(screen.queryByText('React Hooks')).not.toBeInTheDocument();
  expect(screen.getByText('Vue Composition API')).toBeInTheDocument();
});
```

### Clipboard and Selection

```typescript
it('copies text to clipboard', async () => {
  const user = userEvent.setup();
  render(<CopyButton text="secret-token-123" />);

  await user.click(screen.getByRole('button', { name: 'Copy' }));
  const clipboardText = await navigator.clipboard.readText();
  expect(clipboardText).toBe('secret-token-123');
});
```

## Async Testing

### Waiting for Data

Use `findBy` queries, which combine `getBy` with `waitFor`. They retry until the element appears or the timeout expires.

```typescript
it('loads user list from API', async () => {
  render(<UserList />);

  // Wait for loading to complete
  const users = await screen.findAllByRole('listitem');
  expect(users).toHaveLength(3);
});
```

### waitFor for Complex Assertions

Use `waitFor` when the assertion itself needs to eventually pass but there is no single element to wait for.

```typescript
it('removes item after delete confirmation', async () => {
  const user = userEvent.setup();
  render(<TodoList />);

  await user.click(screen.getByRole('button', { name: 'Delete "Buy milk"' }));
  await user.click(screen.getByRole('button', { name: 'Confirm' }));

  await waitFor(() => {
    expect(screen.queryByText('Buy milk')).not.toBeInTheDocument();
    expect(screen.getByText('2 items remaining')).toBeInTheDocument();
  });
});
```

### waitForElementToBeRemoved

```typescript
it('hides loading spinner after data loads', async () => {
  render(<Dashboard />);

  expect(screen.getByRole('progressbar')).toBeInTheDocument();
  await waitForElementToBeRemoved(() => screen.queryByRole('progressbar'));
  expect(screen.getByRole('heading', { name: 'Dashboard' })).toBeInTheDocument();
});
```

## Snapshot Testing

Use snapshots sparingly and only for stable UI structures. Prefer explicit assertions over snapshots.

```typescript
// Acceptable: Small, stable UI component
it('renders badge variants correctly', () => {
  const { container } = render(
    <div>
      <Badge variant="success">Active</Badge>
      <Badge variant="warning">Pending</Badge>
      <Badge variant="error">Failed</Badge>
    </div>
  );

  expect(container).toMatchSnapshot();
});

// Better: Inline snapshots for small output
it('renders formatted date', () => {
  render(<FormattedDate date={new Date('2024-01-15')} />);
  expect(screen.getByText('January 15, 2024')).toMatchInlineSnapshot();
});
```

Avoid snapshots for large component trees, dynamic content, or frequently changing UI. They become noise rather than signal.

## Integration Tests

Test complete user flows that span multiple components.

```typescript
describe('Checkout flow', () => {
  it('completes a purchase from cart to confirmation', async () => {
    const user = userEvent.setup();
    renderWithProviders(<App />, { initialRoute: '/cart' });

    // Cart page
    expect(screen.getByRole('heading', { name: 'Your Cart' })).toBeInTheDocument();
    expect(screen.getByText('2 items')).toBeInTheDocument();
    await user.click(screen.getByRole('button', { name: 'Proceed to checkout' }));

    // Shipping page
    await user.type(screen.getByLabelText('Address'), '123 Main St');
    await user.type(screen.getByLabelText('City'), 'Springfield');
    await user.selectOptions(screen.getByLabelText('State'), 'IL');
    await user.type(screen.getByLabelText('ZIP'), '62701');
    await user.click(screen.getByRole('button', { name: 'Continue to payment' }));

    // Payment page
    await user.type(screen.getByLabelText('Card number'), '4242424242424242');
    await user.type(screen.getByLabelText('Expiry'), '12/25');
    await user.type(screen.getByLabelText('CVC'), '123');
    await user.click(screen.getByRole('button', { name: 'Place order' }));

    // Confirmation
    expect(await screen.findByRole('heading', { name: 'Order Confirmed' })).toBeInTheDocument();
    expect(screen.getByText(/order #/i)).toBeInTheDocument();
  });
});
```

## MSW for API Mocking

### Handler Organization

Organize handlers by domain and compose them.

```typescript
// mocks/handlers/users.ts
export const userHandlers = [
  http.get('/api/users', () => {
    return HttpResponse.json(mockUsers);
  }),
  http.get('/api/users/:id', ({ params }) => {
    const user = mockUsers.find((u) => u.id === params.id);
    if (!user) return new HttpResponse(null, { status: 404 });
    return HttpResponse.json(user);
  }),
  http.patch('/api/users/:id', async ({ params, request }) => {
    const updates = await request.json();
    return HttpResponse.json({ ...mockUsers[0], ...updates });
  }),
];

// mocks/handlers/index.ts
import { userHandlers } from './users';
import { productHandlers } from './products';
import { authHandlers } from './auth';

export const handlers = [
  ...authHandlers,
  ...userHandlers,
  ...productHandlers,
];
```

### Network Error Simulation

```typescript
it('handles network failures gracefully', async () => {
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.error(); // Simulate network error
    })
  );

  render(<UserList />);

  expect(await screen.findByRole('alert')).toHaveTextContent(
    'Unable to load users. Please try again.'
  );
});

it('handles slow responses', async () => {
  server.use(
    http.get('/api/users', async () => {
      await delay(5000);
      return HttpResponse.json(mockUsers);
    })
  );

  render(<UserList />);
  expect(screen.getByRole('progressbar')).toBeInTheDocument();
});
```

### Response Delay for Loading States

```typescript
import { delay, http, HttpResponse } from 'msw';

server.use(
  http.get('/api/data', async () => {
    await delay(100); // Short delay to test loading state
    return HttpResponse.json(mockData);
  })
);
```

## Testing Custom Hooks

### Hooks with Dependencies

When a hook depends on a provider, pass a wrapper.

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useUser } from './useUser';

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };
}

describe('useUser', () => {
  it('fetches and returns user data', async () => {
    const { result } = renderHook(() => useUser('123'), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual({
      id: '123',
      name: 'Jane Doe',
      email: 'jane@example.com',
    });
  });
});
```

## Testing Context Providers

Test the provider and consumers together to verify the integration.

```typescript
describe('ThemeProvider', () => {
  it('provides default theme and allows toggling', async () => {
    const user = userEvent.setup();

    function TestConsumer() {
      const { theme, toggleTheme } = useTheme();
      return (
        <div>
          <span data-testid="theme">{theme}</span>
          <button onClick={toggleTheme}>Toggle</button>
        </div>
      );
    }

    render(
      <ThemeProvider>
        <TestConsumer />
      </ThemeProvider>
    );

    expect(screen.getByTestId('theme')).toHaveTextContent('light');
    await user.click(screen.getByRole('button', { name: 'Toggle' }));
    expect(screen.getByTestId('theme')).toHaveTextContent('dark');
  });
});
```

## Testing Error Boundaries

Suppress `console.error` output during error boundary tests since React logs errors during error handling.

```typescript
describe('ErrorBoundary', () => {
  const originalError = console.error;

  beforeEach(() => {
    console.error = vi.fn();
  });

  afterEach(() => {
    console.error = originalError;
  });

  it('renders fallback when child throws', () => {
    function ThrowingComponent(): never {
      throw new Error('Test error');
    }

    render(
      <ErrorBoundary fallback={<div>Something went wrong</div>}>
        <ThrowingComponent />
      </ErrorBoundary>
    );

    expect(screen.getByText('Something went wrong')).toBeInTheDocument();
  });

  it('recovers when reset', async () => {
    const user = userEvent.setup();
    let shouldThrow = true;

    function MaybeThrows() {
      if (shouldThrow) throw new Error('Boom');
      return <div>Recovered</div>;
    }

    render(
      <ErrorBoundary
        FallbackComponent={({ resetErrorBoundary }) => (
          <div>
            <p>Error occurred</p>
            <button onClick={resetErrorBoundary}>Retry</button>
          </div>
        )}
      >
        <MaybeThrows />
      </ErrorBoundary>
    );

    expect(screen.getByText('Error occurred')).toBeInTheDocument();

    shouldThrow = false;
    await user.click(screen.getByRole('button', { name: 'Retry' }));

    expect(screen.getByText('Recovered')).toBeInTheDocument();
  });
});
```
