---
name: TypeScript Testing
description: This skill should be used when the user asks about "test TypeScript", "Vitest", "Jest TypeScript", "type testing", "mock typed dependencies", "tsd", "expect-type", or "TypeScript test setup". It covers testing TypeScript code with Vitest/Jest, type-level testing, and mocking strategies.
---

# TypeScript Testing

## Vitest Setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      exclude: ['**/*.d.ts', '**/*.test.ts', '**/types/**'],
    },
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') },
  },
});
```

## Test Patterns

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';

describe('UserService', () => {
  let service: UserService;
  let mockRepo: MockedObject<UserRepository>;

  beforeEach(() => {
    mockRepo = {
      findById: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
    };
    service = new UserService(mockRepo);
  });

  it('should return user by id', async () => {
    const user: User = { id: '1', name: 'Alice', email: 'alice@example.com' };
    mockRepo.findById.mockResolvedValue(user);

    const result = await service.getUser('1');

    expect(result).toEqual(user);
    expect(mockRepo.findById).toHaveBeenCalledWith('1');
  });

  it('should throw on not found', async () => {
    mockRepo.findById.mockResolvedValue(null);

    await expect(service.getUser('999')).rejects.toThrow('User not found');
  });
});
```

## Mocking Typed Dependencies

```typescript
import { vi, type MockedFunction } from 'vitest';

// Mock a module
vi.mock('@/services/email', () => ({
  sendEmail: vi.fn(),
}));

import { sendEmail } from '@/services/email';
const mockedSendEmail = sendEmail as MockedFunction<typeof sendEmail>;

// Mock with implementation
mockedSendEmail.mockImplementation(async (to, subject) => {
  return { success: true, messageId: 'mock-id' };
});

// Type-safe mock objects
function createMock<T>(overrides: Partial<T> = {}): T {
  return overrides as T;
}

const mockLogger = createMock<Logger>({
  info: vi.fn(),
  error: vi.fn(),
});
```

## Type-Level Testing

```typescript
// Using expectTypeOf (built into Vitest)
import { expectTypeOf } from 'vitest';

it('should have correct return type', () => {
  expectTypeOf(getUser).returns.toMatchTypeOf<Promise<User>>();
  expectTypeOf(getUser).parameter(0).toBeString();
});

it('should infer generic correctly', () => {
  const result = createPair('hello', 42);
  expectTypeOf(result).toEqualTypeOf<[string, number]>();
});

// Using tsd (for library type testing)
// test-d/index.test-d.ts
import { expectType, expectError } from 'tsd';
import { createStore } from '../src';

const store = createStore({ count: 0 });
expectType<number>(store.getState().count);
expectError(store.getState().nonExistent);
```

## Testing Zod Schemas

```typescript
describe('UserSchema', () => {
  it('should accept valid data', () => {
    const valid = { id: '550e8400-e29b-41d4-a716-446655440000', email: 'test@example.com', name: 'Alice', role: 'user' };
    expect(UserSchema.safeParse(valid).success).toBe(true);
  });

  it('should reject invalid email', () => {
    const invalid = { id: '1', email: 'not-an-email', name: 'Alice', role: 'user' };
    const result = UserSchema.safeParse(invalid);
    expect(result.success).toBe(false);
    if (!result.success) {
      expect(result.error.issues[0].path).toEqual(['email']);
    }
  });

  it('should reject unknown roles', () => {
    const invalid = { id: '1', email: 'test@example.com', name: 'Alice', role: 'superadmin' };
    expect(UserSchema.safeParse(invalid).success).toBe(false);
  });
});
```

## Jest with TypeScript

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' },
  collectCoverageFrom: ['src/**/*.ts', '!src/**/*.d.ts', '!src/**/*.test.ts'],
};

export default config;
```

## References

- [Testing Utilities](references/testing-utilities.md) — Custom matchers, test factories, snapshot testing, parametrized tests.
- [Mocking Strategies](references/mocking-strategies.md) — Module mocks, class mocks, dependency injection for testing, MSW.
