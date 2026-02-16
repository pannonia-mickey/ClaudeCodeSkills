---
name: TypeScript Tooling
description: This skill should be used when the user asks about "tsconfig", "TypeScript configuration", "project references", "path aliases", "module resolution", "esbuild TypeScript", "SWC", "declaration files", "d.ts", "TypeScript build", or "monorepo TypeScript". It covers tsconfig strategies, build tools, module resolution, and project setup.
---

# TypeScript Tooling & Configuration

## tsconfig.json Strategies

### Strict Recommended Base

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### Node.js Backend Config

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "sourceMap": true
  }
}
```

### Library Config (Dual CJS/ESM)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

## Project References (Monorepo)

```
packages/
  shared/      # tsconfig.json + src/
  api/         # tsconfig.json + src/ (depends on shared)
  web/         # tsconfig.json + src/ (depends on shared)
tsconfig.json  # Root solution file
```

```json
// Root tsconfig.json
{
  "files": [],
  "references": [
    { "path": "packages/shared" },
    { "path": "packages/api" },
    { "path": "packages/web" }
  ]
}

// packages/api/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "references": [{ "path": "../shared" }]
}
```

```bash
# Build with project references
tsc --build                     # Build all
tsc --build --watch            # Watch mode
tsc --build --clean            # Clean outputs
```

## Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

Bundler-specific alias resolution:
```typescript
// vite.config.ts
export default defineConfig({
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') },
  },
});
```

## Build Tools

### esbuild (Fastest, type-stripping only)

```bash
esbuild src/index.ts --bundle --platform=node --outdir=dist --format=esm
```

### SWC (Fast, Rust-based)

```json
// .swcrc
{
  "jsc": {
    "parser": { "syntax": "typescript", "tsx": true, "decorators": true },
    "transform": { "legacyDecorator": true },
    "target": "es2022"
  },
  "module": { "type": "es6" }
}
```

### tsx (Dev runner, no build step)

```bash
npx tsx src/index.ts         # Run TS directly
npx tsx watch src/index.ts   # Watch mode
```

## References

- [Module Resolution](references/module-resolution.md) — ESM vs CJS, node16 vs bundler, package.json exports, conditional exports.
- [Migration Guide](references/migration-guide.md) — JS to TS migration strategies, incremental adoption, strictness levels.
