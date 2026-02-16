# TypeScript Validation Patterns

## Zod Advanced Patterns

### Discriminated Unions

```typescript
const ShapeSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('circle'), radius: z.number().positive() }),
  z.object({ type: z.literal('rectangle'), width: z.number().positive(), height: z.number().positive() }),
  z.object({ type: z.literal('triangle'), base: z.number().positive(), height: z.number().positive() }),
]);

type Shape = z.infer<typeof ShapeSchema>;
```

### Transforms

```typescript
const DateStringSchema = z.string().datetime().transform(s => new Date(s));
// Input: string, Output: Date

const TrimmedString = z.string().trim().toLowerCase();

const CoerceNumber = z.coerce.number(); // Accepts "42" → 42
```

### Refinements and Superrefine

```typescript
const PasswordSchema = z.string()
  .min(8, 'At least 8 characters')
  .refine(pw => /[A-Z]/.test(pw), 'Must contain uppercase')
  .refine(pw => /[0-9]/.test(pw), 'Must contain number')
  .refine(pw => /[!@#$%]/.test(pw), 'Must contain special character');

const FormSchema = z.object({
  password: z.string(),
  confirmPassword: z.string(),
}).superRefine((data, ctx) => {
  if (data.password !== data.confirmPassword) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Passwords must match',
      path: ['confirmPassword'],
    });
  }
});
```

### Recursive Schemas

```typescript
type Category = { name: string; children: Category[] };

const CategorySchema: z.ZodType<Category> = z.object({
  name: z.string(),
  children: z.lazy(() => CategorySchema.array()),
});
```

### Custom Error Map

```typescript
const customErrorMap: z.ZodErrorMap = (issue, ctx) => {
  if (issue.code === z.ZodIssueCode.invalid_type) {
    if (issue.expected === 'string') return { message: 'This field must be text' };
    if (issue.expected === 'number') return { message: 'This field must be a number' };
  }
  return { message: ctx.defaultError };
};

z.setErrorMap(customErrorMap);
```

## io-ts Patterns

```typescript
import * as t from 'io-ts';
import { isRight } from 'fp-ts/Either';

const User = t.type({
  id: t.string,
  name: t.string,
  email: t.string,
  age: t.union([t.number, t.undefined]),
});

type User = t.TypeOf<typeof User>;

const result = User.decode(externalData);
if (isRight(result)) {
  console.log(result.right); // User
} else {
  console.error(result.left); // Validation errors
}
```

## Valibot (Lightweight Alternative)

```typescript
import * as v from 'valibot';

const UserSchema = v.object({
  id: v.pipe(v.string(), v.uuid()),
  email: v.pipe(v.string(), v.email()),
  name: v.pipe(v.string(), v.minLength(1)),
  role: v.picklist(['admin', 'user']),
});

type User = v.InferOutput<typeof UserSchema>;
const user = v.parse(UserSchema, data); // Throws on invalid
```

## Schema-First API Contract

```typescript
// Define schemas as single source of truth
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  password: z.string().min(8),
});

const UserResponseSchema = z.object({
  id: z.string().uuid(),
  email: z.string(),
  name: z.string(),
  createdAt: z.string().datetime(),
});

// Derive types
type CreateUserInput = z.infer<typeof CreateUserSchema>;
type UserResponse = z.infer<typeof UserResponseSchema>;

// Use in handler
async function createUser(raw: unknown): Promise<UserResponse> {
  const input = CreateUserSchema.parse(raw); // Validate input
  const user = await db.users.create(input);
  return UserResponseSchema.parse(user); // Validate output
}
```
