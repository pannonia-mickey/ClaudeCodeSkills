# TypeScript Type Challenges

## Implement Pick

```typescript
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

## Implement Readonly

```typescript
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};
```

## Tuple to Union

```typescript
type TupleToUnion<T extends readonly unknown[]> = T[number];
type U = TupleToUnion<[string, number, boolean]>; // string | number | boolean
```

## Last of Array

```typescript
type Last<T extends readonly unknown[]> = T extends [...unknown[], infer L] ? L : never;
type L = Last<[1, 2, 3]>; // 3
```

## Chainable Options

```typescript
type Chainable<T = {}> = {
  option<K extends string, V>(
    key: K extends keyof T ? never : K,
    value: V
  ): Chainable<T & Record<K, V>>;
  get(): T;
};
```

## Deep Get by Path

```typescript
type Get<T, Path extends string> =
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? Get<T[Key], Rest>
      : never
    : Path extends keyof T
    ? T[Path]
    : never;

type Val = Get<{ a: { b: { c: number } } }, 'a.b.c'>; // number
```

## String to Union

```typescript
type StringToUnion<S extends string> =
  S extends `${infer C}${infer Rest}`
    ? C | StringToUnion<Rest>
    : never;

type SU = StringToUnion<'hello'>; // 'h' | 'e' | 'l' | 'o'
```

## Type-Safe Object.keys

```typescript
function typedKeys<T extends object>(obj: T): (keyof T)[] {
  return Object.keys(obj) as (keyof T)[];
}

// Type-safe Object.entries
function typedEntries<T extends Record<string, unknown>>(
  obj: T
): [keyof T, T[keyof T]][] {
  return Object.entries(obj) as [keyof T, T[keyof T]][];
}
```

## Permutation Type

```typescript
type Permutation<T, U = T> = [T] extends [never]
  ? []
  : T extends T
  ? [T, ...Permutation<Exclude<U, T>>]
  : never;

type P = Permutation<'a' | 'b'>; // ['a', 'b'] | ['b', 'a']
```

## IsUnion Detection

```typescript
type IsUnion<T, U = T> = T extends T
  ? [U] extends [T]
    ? false
    : true
  : never;

type Test1 = IsUnion<string>;           // false
type Test2 = IsUnion<string | number>;  // true
```

## Type-Safe Merge

```typescript
type Merge<A, B> = {
  [K in keyof A | keyof B]: K extends keyof B
    ? B[K]
    : K extends keyof A
    ? A[K]
    : never;
};

type Merged = Merge<{ a: string; b: number }, { b: string; c: boolean }>;
// { a: string; b: string; c: boolean }
```
