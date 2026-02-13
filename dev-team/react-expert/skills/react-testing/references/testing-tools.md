# React Testing Tools

## RTL Query Priority

React Testing Library provides multiple query types. Use them in this priority order, from most to least preferred.

### 1. Accessible Roles (Highest Priority)

Queries that reflect how assistive technology and users perceive the page.

```typescript
screen.getByRole('button', { name: 'Submit' })
screen.getByRole('heading', { name: 'Dashboard', level: 2 })
screen.getByRole('textbox', { name: 'Email' })
screen.getByRole('checkbox', { name: 'Remember me' })
screen.getByRole('combobox', { name: 'Country' })
screen.getByRole('tab', { name: 'Settings', selected: true })
screen.getByRole('alert')
screen.getByRole('dialog', { name: 'Confirm deletion' })
screen.getByRole('navigation', { name: 'Main' })
```

### 2. Label Text

For form elements with associated labels.

```typescript
screen.getByLabelText('Email address')
screen.getByLabelText('Password')
```

### 3. Placeholder Text

When no label is available (rare, usually indicates an accessibility problem).

```typescript
screen.getByPlaceholderText('Search...')
```

### 4. Text Content

For non-interactive elements where role queries do not apply.

```typescript
screen.getByText('Welcome to the dashboard')
screen.getByText(/no results found/i)
```

### 5. Display Value

For inputs with a current value.

```typescript
screen.getByDisplayValue('user@example.com')
```

### 6. Alt Text

For images.

```typescript
screen.getByAltText('Company logo')
```

### 7. Title Attribute

Rarely needed; prefer role or text queries.

```typescript
screen.getByTitle('Close')
```

### 8. Test ID (Lowest Priority)

Use only when no other query works. Indicates a possible accessibility gap.

```typescript
screen.getByTestId('complex-chart-container')
```

## Query Variants

Each query comes in three variants:

| Variant | Returns | Throws on Missing | Async |
|---|---|---|---|
| `getBy` | Element | Yes | No |
| `queryBy` | Element or `null` | No | No |
| `findBy` | Promise<Element> | Yes | Yes |

Use `getBy` for elements that must exist. Use `queryBy` when asserting an element does NOT exist. Use `findBy` for elements that appear asynchronously.

```typescript
// Element must exist now
const button = screen.getByRole('button', { name: 'Save' });

// Assert element does not exist
expect(screen.queryByRole('alert')).not.toBeInTheDocument();

// Wait for element to appear
const successMessage = await screen.findByText('Saved successfully');
```

## Custom Render with Providers

### Full Application Wrapper

```typescript
// test/render.tsx
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ThemeProvider } from '@/providers/ThemeProvider';
import { AuthProvider } from '@/providers/AuthProvider';
import { MemoryRouter } from 'react-router-dom';
import { ReactElement } from 'react';

interface ExtendedRenderOptions extends Omit<RenderOptions, 'wrapper'> {
  initialRoute?: string;
  queryClient?: QueryClient;
  user?: User | null;
}

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: 0,
        staleTime: 0,
      },
      mutations: {
        retry: false,
      },
    },
    logger: {
      log: console.log,
      warn: console.warn,
      error: () => {}, // Suppress error logging in tests
    },
  });
}

export function renderApp(
  ui: ReactElement,
  {
    initialRoute = '/',
    queryClient = createTestQueryClient(),
    user = null,
    ...renderOptions
  }: ExtendedRenderOptions = {}
) {
  function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        <AuthProvider initialUser={user}>
          <ThemeProvider>
            <MemoryRouter initialEntries={[initialRoute]}>
              {children}
            </MemoryRouter>
          </ThemeProvider>
        </AuthProvider>
      </QueryClientProvider>
    );
  }

  return {
    ...render(ui, { wrapper: Wrapper, ...renderOptions }),
    queryClient,
  };
}

// Re-export everything from RTL for convenience
export { screen, waitFor, waitForElementToBeRemoved, within, act } from '@testing-library/react';
export { default as userEvent } from '@testing-library/user-event';
```

### Partial Provider Wrappers

For unit tests that need only specific providers:

```typescript
export function renderWithQuery(ui: ReactElement, options?: ExtendedRenderOptions) {
  const queryClient = options?.queryClient ?? createTestQueryClient();

  function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  }

  return { ...render(ui, { wrapper: Wrapper }), queryClient };
}

export function renderWithRouter(ui: ReactElement, initialRoute = '/') {
  return render(ui, {
    wrapper: ({ children }) => (
      <MemoryRouter initialEntries={[initialRoute]}>
        {children}
      </MemoryRouter>
    ),
  });
}
```

## Test Utilities

### Custom Matchers

Extend Vitest/Jest matchers for common assertions.

```typescript
// test/matchers.ts
expect.extend({
  toBeWithinRange(received: number, floor: number, ceiling: number) {
    const pass = received >= floor && received <= ceiling;
    return {
      pass,
      message: () =>
        `expected ${received} to be within range ${floor} - ${ceiling}`,
    };
  },
});
```

### Factory Functions for Test Data

Create factories instead of duplicating mock data across tests.

```typescript
// test/factories.ts
let idCounter = 0;

export function createUser(overrides: Partial<User> = {}): User {
  idCounter += 1;
  return {
    id: `user-${idCounter}`,
    name: `Test User ${idCounter}`,
    email: `user${idCounter}@example.com`,
    role: 'member',
    createdAt: new Date().toISOString(),
    ...overrides,
  };
}

export function createProduct(overrides: Partial<Product> = {}): Product {
  idCounter += 1;
  return {
    id: `product-${idCounter}`,
    name: `Product ${idCounter}`,
    price: 9.99 + idCounter,
    category: 'general',
    inStock: true,
    ...overrides,
  };
}

// Usage in tests
const admin = createUser({ role: 'admin', name: 'Admin User' });
const outOfStockItem = createProduct({ inStock: false, name: 'Rare Widget' });
```

### Screen Debug Helpers

```typescript
// Debug the entire screen
screen.debug();

// Debug a specific element
screen.debug(screen.getByRole('form'));

// Log accessible roles for the entire document
screen.logTestingPlaygroundURL();
```

## Coverage

### Configuration

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      include: ['src/**/*.{ts,tsx}'],
      exclude: [
        'src/**/*.d.ts',
        'src/**/*.test.{ts,tsx}',
        'src/**/*.stories.{ts,tsx}',
        'src/test/**',
        'src/mocks/**',
        'src/**/index.ts', // barrel files
      ],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      },
    },
  },
});
```

### What to Cover

Prioritize coverage of:
- Business logic and utility functions (aim for 90%+)
- User-facing components with interactions (aim for 80%+)
- Custom hooks (aim for 90%+)
- Error handling paths

Skip coverage for:
- Third-party library wrappers with no logic
- Purely presentational components (test with Storybook instead)
- Generated code and type definitions

## Vitest vs Jest

### Key Differences

| Feature | Vitest | Jest |
|---|---|---|
| Config | Native ESM, Vite-powered | CommonJS, custom transform |
| Speed | Faster with HMR and native ESM | Slower on large projects |
| TypeScript | Native support via Vite | Requires ts-jest or @swc/jest |
| API Compatibility | Jest-compatible API | N/A |
| Watch Mode | Instant via Vite HMR | File system polling |
| Module Mocking | `vi.mock()` with hoisting | `jest.mock()` with hoisting |
| Browser Mode | Built-in via @vitest/browser | Requires separate setup |

### Migration from Jest to Vitest

```typescript
// Replace imports
// jest.fn() -> vi.fn()
// jest.mock() -> vi.mock()
// jest.spyOn() -> vi.spyOn()
// jest.useFakeTimers() -> vi.useFakeTimers()

// Update config
// jest.config.js -> vitest.config.ts
// Remove babel-jest, ts-jest transforms
// Remove moduleNameMapper (use Vite resolve.alias instead)
```

### Vitest Configuration for React

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./test/setup.ts'],
    include: ['src/**/*.test.{ts,tsx}'],
    css: { modules: { classNameStrategy: 'non-scoped' } },
  },
});
```

## @testing-library/user-event v14

### Setup

Always call `userEvent.setup()` before interactions. This creates a user instance that tracks keyboard state and pointer position.

```typescript
const user = userEvent.setup();
```

### Advanced Options

```typescript
const user = userEvent.setup({
  delay: null, // Disable delay for faster tests (default: 0)
  advanceTimers: vi.advanceTimersByTime, // Integrate with fake timers
  pointerEventsCheck: 0, // Skip pointer-events check (0 = disable)
});
```

### API Reference

```typescript
// Typing
await user.type(element, 'hello world');
await user.type(element, '{Enter}'); // Special keys
await user.clear(element); // Clear input

// Clicking
await user.click(element);
await user.dblClick(element);
await user.tripleClick(element); // Select all text in input

// Keyboard
await user.keyboard('{Shift>}A{/Shift}'); // Hold shift, press A
await user.keyboard('{Control>}c{/Control}'); // Ctrl+C
await user.tab(); // Tab to next focusable element
await user.tab({ shift: true }); // Shift+Tab

// Selection
await user.selectOptions(selectElement, ['value1', 'value2']);
await user.deselectOptions(selectElement, ['value1']);

// Clipboard
await user.copy();
await user.cut();
await user.paste('pasted text');

// Pointer
await user.hover(element);
await user.unhover(element);
await user.pointer({ target: element, offset: 5 }); // Click at offset
```

### Key Differences from v13

- `setup()` is now required (no more static methods)
- All methods are async
- `type` dispatches real keyboard events including keyDown, keyPress, keyUp
- Built-in fake timer integration via `advanceTimers` option
- Pointer API replaces the old `click` internals with a full pointer simulation model
