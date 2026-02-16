---
name: TypeScript Type System
description: This skill should be used when the user asks about "conditional types", "mapped types", "template literal types", "recursive types", "type-level programming", "infer keyword", "variance annotations", "covariance", "contravariance", or "advanced TypeScript types". It covers advanced type system features for type-level computation and complex type transformations.
---

# TypeScript Advanced Type System

## Conditional Types

```typescript
// Basic conditional
type IsString<T> = T extends string ? true : false;
type A = IsString<'hello'>; // true
type B = IsString<42>;      // false

// Distributed conditional (applies to each union member)
type ToArray<T> = T extends unknown ? T[] : never;
type C = ToArray<string | number>; // string[] | number[]

// Non-distributive (wrapping in tuple prevents distribution)
type ToArrayND<T> = [T] extends [unknown] ? T[] : never;
type D = ToArrayND<string | number>; // (string | number)[]

// infer — extract types from structures
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never;
type UnpackPromise<T> = T extends Promise<infer U> ? UnpackPromise<U> : T;
type ArrayItem<T> = T extends readonly (infer U)[] ? U : T;

// infer with constraints (TS 4.7+)
type FirstString<T> = T extends [infer S extends string, ...unknown[]] ? S : never;
```

## Mapped Types

```typescript
// Transform all properties
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Key remapping (TS 4.1+)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
// { getName: () => string; getAge: () => number; }

// Filter keys by value type
type StringKeys<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

// Make specific keys required, rest optional
type RequireKeys<T, K extends keyof T> = Required<Pick<T, K>> & Partial<Omit<T, K>>;
```

## Template Literal Types

```typescript
// String manipulation at the type level
type EventName<T extends string> = `on${Capitalize<T>}`;
type E = EventName<'click'>; // "onClick"

// Pattern matching
type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParams<Rest>
    : T extends `${string}:${infer Param}`
    ? Param
    : never;

type Params = ExtractParams<'/users/:userId/posts/:postId'>; // "userId" | "postId"

// CSS unit types
type CSSUnit = 'px' | 'rem' | 'em' | '%' | 'vh' | 'vw';
type CSSValue = `${number}${CSSUnit}`;
const width: CSSValue = '100px'; // OK
// const bad: CSSValue = '100'; // Error

// Dot-path accessor
type DotPath<T, Prefix extends string = ''> = T extends object
  ? { [K in keyof T & string]:
      | `${Prefix}${K}`
      | DotPath<T[K], `${Prefix}${K}.`>
    }[keyof T & string]
  : never;

type UserPaths = DotPath<{ name: string; address: { city: string; zip: string } }>;
// "name" | "address" | "address.city" | "address.zip"
```

## Recursive Types

```typescript
// Deep partial
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Deep readonly
type DeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

// JSON-safe types
type JsonValue = string | number | boolean | null | JsonValue[] | { [key: string]: JsonValue };

// Flatten nested arrays
type Flatten<T> = T extends readonly (infer U)[] ? Flatten<U> : T;
type F = Flatten<number[][][]>; // number
```

## Variance Annotations (TS 4.7+)

```typescript
// Explicit variance for type parameters
interface Producer<out T> {   // covariant — T only in output position
  get(): T;
}

interface Consumer<in T> {    // contravariant — T only in input position
  accept(value: T): void;
}

interface Transformer<in T, out U> {  // T contravariant, U covariant
  transform(input: T): U;
}

// Invariant — T in both positions (default)
interface Container<T> {
  get(): T;
  set(value: T): void;
}
```

## References

- [Type Challenges](references/type-challenges.md) — Practical type-level programming exercises and solutions.
- [Type Patterns Library](references/type-patterns-library.md) — Reusable utility type patterns for common scenarios.
