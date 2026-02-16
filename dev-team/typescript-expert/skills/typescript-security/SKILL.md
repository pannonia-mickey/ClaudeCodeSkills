---
name: TypeScript Security
description: This skill should be used when the user asks about "runtime validation", "Zod schema", "io-ts", "type assertion abuse", "TypeScript security", "validate API response", "type-safe parsing", or "secure TypeScript patterns". It covers using the type system for safety, runtime validation libraries, and avoiding common type safety pitfalls.
---

# TypeScript Security Patterns

## Runtime Validation with Zod

```typescript
import { z } from 'zod';

// Define schema — source of truth for both runtime and compile-time
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(['admin', 'user', 'moderator']),
  age: z.number().int().min(13).max(120).optional(),
  metadata: z.record(z.unknown()).optional(),
});

// Infer TypeScript type from schema
type User = z.infer<typeof UserSchema>;

// Parse with validation (throws on invalid)
const user = UserSchema.parse(externalData);

// Safe parse (returns result object)
const result = UserSchema.safeParse(externalData);
if (result.success) {
  console.log(result.data); // User
} else {
  console.error(result.error.flatten());
}
```

## Environment Variable Validation

```typescript
const EnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  API_KEY: z.string().min(10),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  ENABLE_CACHE: z.coerce.boolean().default(false),
});

// Validate at startup — fail fast
export const env = EnvSchema.parse(process.env);
// env.PORT is number (not string)
// env.NODE_ENV is 'development' | 'production' | 'test'
```

## API Response Validation

```typescript
const ApiResponseSchema = z.object({
  data: z.array(UserSchema),
  pagination: z.object({
    page: z.number(),
    pageSize: z.number(),
    total: z.number(),
  }),
});

async function fetchUsers(): Promise<z.infer<typeof ApiResponseSchema>> {
  const response = await fetch('/api/users');
  const json = await response.json();
  return ApiResponseSchema.parse(json); // Runtime validation
}
```

## Avoiding Type Assertion Abuse

```typescript
// BAD — Type assertion hides errors
const user = apiResponse as User; // No runtime validation!
const config = {} as Config; // Lies to the compiler

// GOOD — Parse and validate
const user = UserSchema.parse(apiResponse);

// GOOD — Type guard with validation
function isUser(value: unknown): value is User {
  return UserSchema.safeParse(value).success;
}

// ACCEPTABLE — assertion after validation
const data: unknown = JSON.parse(rawString);
const validated = UserSchema.parse(data); // Throws if invalid
```

## Branded Types for Domain Safety

```typescript
type Email = string & { readonly __brand: 'Email' };
type SqlSafe = string & { readonly __brand: 'SqlSafe' };

function validateEmail(input: string): Email {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input)) {
    throw new Error('Invalid email');
  }
  return input as Email;
}

function sanitizeSql(input: string): SqlSafe {
  // Proper parameterized queries are better, but this prevents mixing
  return input.replace(/['";]/g, '') as SqlSafe;
}

// Functions require branded types — can't pass raw strings
function sendEmail(to: Email, subject: string) { /* ... */ }
sendEmail(validateEmail('user@example.com'), 'Hello'); // OK
// sendEmail('raw@string.com', 'Hello'); // Error
```

## References

- [Validation Patterns](references/validation-patterns.md) — Zod advanced patterns, discriminated unions, transforms, custom error maps.
- [Secure Coding](references/secure-coding.md) — Type-safe SQL, XSS prevention, secrets handling, dependency auditing.
