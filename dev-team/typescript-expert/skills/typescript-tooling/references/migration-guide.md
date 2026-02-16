# JavaScript to TypeScript Migration

## Phased Migration Strategy

### Phase 1: Setup (Day 1)

```json
// tsconfig.json — Start permissive
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "strict": false,
    "noEmit": true,
    "moduleResolution": "bundler",
    "module": "ESNext",
    "target": "ES2022",
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src"]
}
```

### Phase 2: Enable checkJs

```json
{
  "compilerOptions": {
    "checkJs": true,
    "allowJs": true
  }
}
```

Use JSDoc annotations in JS files for gradual typing:
```javascript
/** @type {import('./types').User} */
const user = getUser();

/** @param {string} name @returns {boolean} */
function isValid(name) { return name.length > 0; }
```

### Phase 3: Rename Files (.js to .ts)

Start with:
1. Leaf files (no dependents) — utilities, helpers
2. Type definition files — create `types.ts` for shared types
3. Core business logic
4. Entry points last

```bash
# Track migration progress
find src -name "*.js" | wc -l   # Remaining JS files
find src -name "*.ts" | wc -l   # Converted TS files
```

### Phase 4: Enable Strict Flags Incrementally

```json
// Enable one at a time, fix errors, then add next
{
  "compilerOptions": {
    // Round 1
    "noImplicitAny": true,

    // Round 2
    "strictNullChecks": true,

    // Round 3
    "strictFunctionTypes": true,
    "strictBindCallApply": true,

    // Round 4 — replace all above with:
    "strict": true,

    // Round 5 — bonus strictness
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

## Handling Untyped Dependencies

```bash
# Check if types exist
npm info @types/lodash

# Install types
npm install -D @types/lodash @types/express
```

If no types exist, create a declaration file:
```typescript
// src/types/untyped-lib.d.ts
declare module 'untyped-lib' {
  export function doSomething(input: string): Promise<Result>;
  export interface Result {
    data: unknown;
    status: number;
  }
}
```

Quick escape hatch (use sparingly):
```typescript
// src/types/untyped-modules.d.ts
declare module 'some-untyped-package';
// This makes all imports `any` — replace with proper types later
```

## Common Migration Patterns

### Implicit Any to Explicit Types

```typescript
// Before (JS)
function processItems(items) {
  return items.map(item => item.name);
}

// After (TS)
interface Item { name: string; id: number; }

function processItems(items: Item[]): string[] {
  return items.map(item => item.name);
}
```

### Null Checks with strictNullChecks

```typescript
// Before — works in JS, fails with strictNullChecks
const user = getUser();
console.log(user.name);

// After
const user = getUser();
if (!user) throw new Error('User not found');
console.log(user.name);

// Or optional chaining
console.log(user?.name ?? 'Unknown');
```

### this Context

```typescript
// Before — `this` type is implicit
class Counter {
  count = 0;
  increment() { this.count++; }
}

// Potential issue when passing as callback
const counter = new Counter();
button.addEventListener('click', counter.increment); // `this` is wrong!

// Fix: arrow function or explicit this
class Counter {
  count = 0;
  increment = () => { this.count++; }; // Arrow preserves this
}
```

## CI Type Coverage Tracking

```bash
# Install type-coverage
npm install -D type-coverage

# Check current coverage
npx type-coverage --strict --detail

# Add to CI
npx type-coverage --at-least 90 --strict
```

```yaml
# GitHub Actions step
- name: Type coverage
  run: npx type-coverage --at-least 95 --strict
```

## @ts-expect-error vs @ts-ignore

```typescript
// @ts-expect-error — use this (errors if the next line has NO error)
// @ts-expect-error: Legacy code, tracked in JIRA-123
const result = legacyFunction(untypedArg);

// @ts-ignore — avoid (silently hides all errors, even new ones)
// Don't use @ts-ignore in new code

// Suppress specific errors with eslint
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const legacy: any = getFromLegacySystem();
```
