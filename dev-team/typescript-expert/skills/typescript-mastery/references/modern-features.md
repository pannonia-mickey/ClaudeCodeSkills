# TypeScript 5+ Modern Features

## Decorators (Stage 3, TS 5.0+)

```typescript
// Standard TC39 decorators (not experimental)
function logged<T extends (...args: any[]) => any>(
  target: T,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: Parameters<T>): ReturnType<T> {
    console.log(`Calling ${String(context.name)} with`, args);
    const result = target.apply(this, args);
    console.log(`${String(context.name)} returned`, result);
    return result;
  } as T;
}

class Calculator {
  @logged
  add(a: number, b: number): number {
    return a + b;
  }
}

// Class decorator
function sealed(target: Function, context: ClassDecoratorContext) {
  Object.seal(target);
  Object.seal(target.prototype);
}

// Field decorator with accessor
function observable<T>(
  _target: undefined,
  context: ClassFieldDecoratorContext<any, T>
) {
  return function (this: any, initialValue: T): T {
    // Intercept field initialization
    return initialValue;
  };
}
```

## Const Type Parameters (TS 5.0+)

```typescript
// Without const: T inferred as string[]
function routes<T extends readonly string[]>(paths: T): T {
  return paths;
}
const r1 = routes(['/', '/about']); // string[]

// With const: T preserves literal types
function routesConst<const T extends readonly string[]>(paths: T): T {
  return paths;
}
const r2 = routesConst(['/', '/about']); // readonly ["/", "/about"]

// Practical use: type-safe configuration
function defineConfig<const T extends Record<string, { type: string; required?: boolean }>>(
  schema: T
): T {
  return schema;
}

const schema = defineConfig({
  name: { type: 'string', required: true },
  age: { type: 'number' },
});
// schema.name.required is true (not boolean | undefined)
```

## Import Attributes (TS 5.3+)

```typescript
// Import JSON with type assertion
import config from './config.json' with { type: 'json' };

// Dynamic import with attributes
const data = await import('./data.json', { with: { type: 'json' } });
```

## Using Declarations (TS 5.2+)

```typescript
// Automatic resource disposal (Symbol.dispose)
class DatabaseConnection implements Disposable {
  [Symbol.dispose]() {
    console.log('Connection closed');
  }
}

class FileHandle implements AsyncDisposable {
  async [Symbol.asyncDispose]() {
    console.log('File closed');
  }
}

// Resources automatically disposed at end of scope
{
  using conn = new DatabaseConnection();
  // use conn...
} // conn[Symbol.dispose]() called here

// Async version
{
  await using file = new FileHandle();
  // use file...
} // await file[Symbol.asyncDispose]() called here
```

## Satisfies Operator (TS 4.9+)

```typescript
// Validate type without widening
type Color = 'red' | 'green' | 'blue';
type ColorMap = Record<Color, string | number[]>;

const palette = {
  red: '#ff0000',
  green: [0, 255, 0],
  blue: '#0000ff',
} satisfies ColorMap;

// palette.red is string (not string | number[])
// palette.green is number[] (not string | number[])
palette.red.toUpperCase(); // OK — TypeScript knows it's a string
```

## NoInfer Utility Type (TS 5.4+)

```typescript
// Prevent type parameter inference from specific positions
function createFSM<S extends string>(config: {
  initial: NoInfer<S>;
  states: S[];
}) {
  return config;
}

// Without NoInfer, 'idle' would widen S to include any string
createFSM({
  initial: 'idle',
  states: ['idle', 'loading', 'done'],
});
// Error if initial is not one of states
```

## Module Resolution: Bundler Mode (TS 5.0+)

```typescript
// tsconfig.json — recommended for modern bundled apps
{
  "compilerOptions": {
    "moduleResolution": "bundler",  // Understands package.json exports
    "module": "esnext",
    "noEmit": true,                 // Let bundler handle emit
    "verbatimModuleSyntax": true,   // Enforce explicit type imports
  }
}

// With verbatimModuleSyntax, must use 'import type' for types
import type { User } from './models'; // Erased at runtime
import { createUser } from './models'; // Kept at runtime
```

## Improved Enums (TS 5.0+)

```typescript
// All enums can now be used as union types
enum Direction {
  Up = 'UP',
  Down = 'DOWN',
  Left = 'LEFT',
  Right = 'RIGHT',
}

// Computed narrowing works
function move(dir: Direction) {
  if (dir === Direction.Up) {
    // dir is Direction.Up
  }
}
```
