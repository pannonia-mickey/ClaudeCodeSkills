# TypeScript Module Resolution

## Module Resolution Strategies

### bundler (Recommended for apps)

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "module": "ESNext"
  }
}
```

- Understands `package.json` `exports` field
- Allows extensionless imports (`import { foo } from './foo'`)
- Best for Vite, webpack, esbuild bundled applications

### NodeNext / Node16 (Recommended for Node.js)

```json
{
  "compilerOptions": {
    "moduleResolution": "NodeNext",
    "module": "NodeNext"
  }
}
```

- Requires file extensions in imports (`import { foo } from './foo.js'`)
- Respects `package.json` `type: "module"` or `"commonjs"`
- `.ts` files must use `.js` extension in imports (maps at compile time)

## Package.json Exports

```json
{
  "name": "my-library",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./utils": {
      "types": "./dist/utils.d.ts",
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.cjs"
    }
  },
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "files": ["dist"]
}
```

Key rules:
- `types` condition must come FIRST in each export entry
- Order matters: TypeScript reads conditions top-to-bottom
- `import` = ESM entry, `require` = CJS entry

## ESM vs CJS Interop

```typescript
// In ESM (type: "module")
import { readFile } from 'node:fs/promises';     // Named import from ESM-compatible
import cjsModule from 'some-cjs-package';         // Default import from CJS
import { createRequire } from 'node:module';

const require = createRequire(import.meta.url);   // For CJS-only packages
const legacy = require('legacy-cjs-only');

// __dirname equivalent in ESM
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

## Verbatim Module Syntax

```json
{
  "compilerOptions": {
    "verbatimModuleSyntax": true
  }
}
```

Enforces:
```typescript
import type { User } from './models';    // Type-only — erased at runtime
import { createUser } from './models';   // Value — kept at runtime

// This is an error with verbatimModuleSyntax:
// import { User } from './models'; // Error if User is only a type

export type { User };                    // Type-only re-export
export { createUser };                   // Value re-export
```

## Declaration Files

```typescript
// global.d.ts — Augment global types
declare global {
  interface Window {
    analytics: AnalyticsClient;
  }

  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      API_KEY: string;
      NODE_ENV: 'development' | 'production' | 'test';
    }
  }
}

export {}; // Make this a module

// module-augmentation.d.ts — Extend third-party types
declare module 'express' {
  interface Request {
    userId?: string;
    permissions?: string[];
  }
}
```

## Barrel Exports

```typescript
// src/models/index.ts
export { User } from './user';
export { Product } from './product';
export { Order } from './order';
export type { UserCreateInput, UserUpdateInput } from './user';

// Usage
import { User, Product } from '@/models';
```

Caveat: Barrel exports can hurt tree-shaking with some bundlers. For libraries, consider direct imports.
