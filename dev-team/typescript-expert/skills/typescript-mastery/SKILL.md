---
name: TypeScript Mastery
description: This skill should be used when the user asks to "write TypeScript", "use generics", "type narrow", "discriminated union", "type guard", "assertion function", "const assertion", "satisfies operator", "utility types", or "TypeScript basics". It covers core TypeScript features, type narrowing, generics, utility types, and modern TS 5+ syntax.
---

# TypeScript Core Features

## Type Narrowing

```typescript
// typeof narrowing
function process(value: string | number) {
  if (typeof value === 'string') {
    return value.toUpperCase(); // string
  }
  return value.toFixed(2); // number
}

// Discriminated unions
type Result<T> = { ok: true; data: T } | { ok: false; error: string };

function handle<T>(result: Result<T>) {
  if (result.ok) {
    console.log(result.data); // T
  } else {
    console.error(result.error); // string
  }
}

// Type predicates
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

// Assertion functions
function assertDefined<T>(value: T | null | undefined, msg?: string): asserts value is T {
  if (value == null) throw new Error(msg ?? 'Value is null or undefined');
}
```

## Generics

```typescript
// Constrained generics
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Generic with default
interface PaginatedResponse<T, Meta = { total: number; page: number }> {
  data: T[];
  meta: Meta;
}

// Generic inference from arguments
function createPair<A, B>(a: A, b: B): [A, B] {
  return [a, b];
}
const pair = createPair('hello', 42); // [string, number]

// Generic class
class TypedMap<K extends string, V> {
  private map = new Map<K, V>();
  set(key: K, value: V) { this.map.set(key, value); }
  get(key: K): V | undefined { return this.map.get(key); }
}
```

## Const Assertions & Satisfies

```typescript
// const assertion — preserves literal types
const ROUTES = {
  home: '/',
  about: '/about',
  products: '/products',
} as const;
// type: { readonly home: "/"; readonly about: "/about"; readonly products: "/products" }

// satisfies — validates type while preserving narrower type
const config = {
  port: 3000,
  host: 'localhost',
  debug: true,
} satisfies Record<string, string | number | boolean>;
// config.port is number (not string | number | boolean)

// Combining both
const COLORS = {
  primary: '#007bff',
  secondary: '#6c757d',
} as const satisfies Record<string, `#${string}`>;
```

## Enums vs Union Types

```typescript
// Prefer union types over enums for most cases
type Status = 'active' | 'inactive' | 'pending';
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';

// Use const object when you need runtime values + type
const LogLevel = {
  DEBUG: 0,
  INFO: 1,
  WARN: 2,
  ERROR: 3,
} as const;
type LogLevel = (typeof LogLevel)[keyof typeof LogLevel]; // 0 | 1 | 2 | 3
```

## Utility Types

```typescript
// Built-in utilities
type UserUpdate = Partial<User>;              // All optional
type RequiredUser = Required<User>;           // All required
type UserName = Pick<User, 'firstName' | 'lastName'>;
type UserWithoutId = Omit<User, 'id'>;
type StringKeys = Record<string, unknown>;
type ActiveStatus = Extract<Status, 'active' | 'pending'>;
type NonActive = Exclude<Status, 'active'>;
type FnReturn = ReturnType<typeof getUser>;
type FnParams = Parameters<typeof getUser>;
type AwaitedUser = Awaited<Promise<User>>;
type ReadonlyUser = Readonly<User>;

// Custom utility: make specific keys optional
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
type CreateUser = PartialBy<User, 'id' | 'createdAt'>;
```

## References

- [Advanced Patterns](references/advanced-patterns.md) — Branded types, builder pattern, exhaustive checking, type-safe error handling.
- [Modern Features](references/modern-features.md) — TypeScript 5+ features: decorators, const type parameters, satisfies, import attributes.
