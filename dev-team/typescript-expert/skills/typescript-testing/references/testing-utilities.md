# TypeScript Testing Utilities

## Custom Matchers (Vitest)

```typescript
// vitest.setup.ts
import { expect } from 'vitest';

expect.extend({
  toBeValidEmail(received: string) {
    const pass = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(received);
    return {
      pass,
      message: () => `expected ${received} ${pass ? 'not ' : ''}to be a valid email`,
    };
  },
  toBeWithinRange(received: number, floor: number, ceiling: number) {
    const pass = received >= floor && received <= ceiling;
    return {
      pass,
      message: () => `expected ${received} to be within range ${floor} - ${ceiling}`,
    };
  },
});

// Type declaration
interface CustomMatchers<R = unknown> {
  toBeValidEmail(): R;
  toBeWithinRange(floor: number, ceiling: number): R;
}

declare module 'vitest' {
  interface Assertion<T = any> extends CustomMatchers<T> {}
  interface AsymmetricMatchersContaining extends CustomMatchers {}
}
```

## Test Factories

```typescript
// factories/user.factory.ts
import { faker } from '@faker-js/faker';

export function createUser(overrides: Partial<User> = {}): User {
  return {
    id: faker.string.uuid(),
    email: faker.internet.email(),
    name: faker.person.fullName(),
    role: 'user',
    createdAt: faker.date.recent().toISOString(),
    ...overrides,
  };
}

export function createUsers(count: number, overrides: Partial<User> = {}): User[] {
  return Array.from({ length: count }, () => createUser(overrides));
}

// Builder pattern for complex objects
class UserBuilder {
  private data: Partial<User> = {};

  withRole(role: User['role']): this {
    this.data.role = role;
    return this;
  }

  withEmail(email: string): this {
    this.data.email = email;
    return this;
  }

  build(): User {
    return createUser(this.data);
  }
}

// Usage
const admin = new UserBuilder().withRole('admin').build();
```

## Snapshot Testing

```typescript
// Value snapshots
it('should serialize config correctly', () => {
  const config = generateConfig({ env: 'production' });
  expect(config).toMatchSnapshot();
});

// Inline snapshots
it('should format error message', () => {
  const error = formatError('NOT_FOUND', 'User');
  expect(error).toMatchInlineSnapshot(`
    {
      "code": "NOT_FOUND",
      "message": "User not found",
      "status": 404,
    }
  `);
});

// Snapshot with dynamic values
it('should create user with id', () => {
  const user = createUser({ name: 'Alice' });
  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(String),
  });
});
```

## Parametrized Tests

```typescript
// Vitest each
it.each([
  { input: 'hello', expected: 'HELLO' },
  { input: 'world', expected: 'WORLD' },
  { input: '', expected: '' },
])('should uppercase "$input" to "$expected"', ({ input, expected }) => {
  expect(input.toUpperCase()).toBe(expected);
});

// Table-driven tests
describe.each([
  { status: 200, expected: true },
  { status: 201, expected: true },
  { status: 400, expected: false },
  { status: 500, expected: false },
])('isSuccessStatus($status)', ({ status, expected }) => {
  it(`should return ${expected}`, () => {
    expect(isSuccessStatus(status)).toBe(expected);
  });
});

// Type-safe test cases
interface TestCase<I, O> { name: string; input: I; expected: O; }

const cases: TestCase<string, number>[] = [
  { name: 'single word', input: 'hello', expected: 1 },
  { name: 'multiple words', input: 'hello world', expected: 2 },
  { name: 'empty string', input: '', expected: 0 },
];

cases.forEach(({ name, input, expected }) => {
  it(`countWords: ${name}`, () => {
    expect(countWords(input)).toBe(expected);
  });
});
```

## Async Testing Patterns

```typescript
// Testing promises
it('should resolve with data', async () => {
  await expect(fetchData()).resolves.toEqual({ id: 1 });
});

it('should reject with error', async () => {
  await expect(fetchInvalidData()).rejects.toThrow('Not found');
});

// Testing timers
import { vi, afterEach } from 'vitest';

beforeEach(() => vi.useFakeTimers());
afterEach(() => vi.useRealTimers());

it('should debounce calls', async () => {
  const fn = vi.fn();
  const debounced = debounce(fn, 300);

  debounced();
  debounced();
  debounced();

  expect(fn).not.toHaveBeenCalled();
  vi.advanceTimersByTime(300);
  expect(fn).toHaveBeenCalledTimes(1);
});
```
