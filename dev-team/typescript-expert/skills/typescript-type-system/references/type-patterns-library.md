# TypeScript Type Patterns Library

## Common Utility Patterns

### Make Specific Keys Optional/Required

```typescript
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
type RequiredBy<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

// Usage
type CreateUser = PartialBy<User, 'id' | 'createdAt'>;
type ValidatedForm = RequiredBy<FormData, 'email' | 'password'>;
```

### Mutable Version of Readonly

```typescript
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
type DeepMutable<T> = T extends object
  ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
  : T;
```

### Non-Nullable Deep

```typescript
type DeepNonNullable<T> = T extends object
  ? { [K in keyof T]: DeepNonNullable<NonNullable<T[K]>> }
  : NonNullable<T>;
```

### Value Type of Object

```typescript
type ValueOf<T> = T[keyof T];
type V = ValueOf<{ a: string; b: number }>; // string | number
```

## Function Patterns

### Overload Helper

```typescript
type Overloads<T extends (...args: any[]) => any> =
  T extends { (...args: infer A1): infer R1; (...args: infer A2): infer R2 }
    ? [(...args: A1) => R1, (...args: A2) => R2]
    : [T];
```

### Async Function Return Type

```typescript
type AsyncReturnType<T extends (...args: any) => Promise<any>> =
  T extends (...args: any) => Promise<infer R> ? R : never;
```

### Typed Event Handler

```typescript
type EventHandler<E extends Record<string, unknown>> = {
  [K in keyof E as `on${Capitalize<string & K>}`]?: (payload: E[K]) => void;
};

type Handlers = EventHandler<{
  click: { x: number; y: number };
  submit: { data: FormData };
}>;
// { onClick?: (payload: { x: number; y: number }) => void; onSubmit?: ... }
```

## Object Patterns

### Path Type for Nested Access

```typescript
type Paths<T, D extends number = 5> = [D] extends [never]
  ? never
  : T extends object
  ? {
      [K in keyof T & string]: T[K] extends object
        ? K | `${K}.${Paths<T[K], Prev[D]>}`
        : K;
    }[keyof T & string]
  : never;

type Prev = [never, 0, 1, 2, 3, 4, 5];
```

### Strict Omit (Errors on Invalid Keys)

```typescript
type StrictOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
// Unlike Omit, this errors if K is not a key of T
```

### Readonly Array Elements

```typescript
type ElementOf<T extends readonly unknown[]> = T[number];
type E = ElementOf<['a', 'b', 'c']>; // "a" | "b" | "c"
```

## API Patterns

### Type-Safe API Routes

```typescript
type ApiRoutes = {
  '/users': { GET: User[]; POST: User };
  '/users/:id': { GET: User; PUT: User; DELETE: void };
  '/products': { GET: Product[] };
};

type ApiMethod<Path extends keyof ApiRoutes> = keyof ApiRoutes[Path];
type ApiResponse<Path extends keyof ApiRoutes, Method extends ApiMethod<Path>> = ApiRoutes[Path][Method];
```

### Form Schema to Validation

```typescript
type FieldConfig = {
  type: 'string' | 'number' | 'boolean';
  required?: boolean;
};

type InferFormValues<T extends Record<string, FieldConfig>> = {
  [K in keyof T]: T[K]['type'] extends 'string'
    ? T[K]['required'] extends true ? string : string | undefined
    : T[K]['type'] extends 'number'
    ? T[K]['required'] extends true ? number : number | undefined
    : T[K]['required'] extends true ? boolean : boolean | undefined;
};
```

## Discriminated Union Helpers

### Match Function

```typescript
type DiscriminatedUnion<K extends string, T extends Record<string, unknown>> = {
  [P in keyof T]: { [Q in K]: P } & T[P];
}[keyof T];

// Pattern matching helper
function match<U extends { type: string }, R>(
  value: U,
  handlers: { [K in U['type']]: (v: Extract<U, { type: K }>) => R }
): R {
  return (handlers as any)[value.type](value);
}

type Action =
  | { type: 'increment'; amount: number }
  | { type: 'reset' }
  | { type: 'set'; value: number };

const result = match(action, {
  increment: ({ amount }) => count + amount,
  reset: () => 0,
  set: ({ value }) => value,
});
```
