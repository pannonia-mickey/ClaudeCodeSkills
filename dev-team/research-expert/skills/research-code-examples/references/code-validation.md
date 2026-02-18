# Code Validation Reference

Systematic steps for verifying that found code examples are correct, current, and safe to adapt before presenting them.

---

## Validation Checklist

Run every found example through this checklist before presenting it.

```
[ ] Version identified — example targets which library version?
[ ] Project version identified — what version does the project use?
[ ] Version compatible — are they the same major version?
[ ] All imports verified — do imported names exist in current API?
[ ] Method signatures verified — parameter names/types/order unchanged?
[ ] Config keys verified — all options in config objects still valid?
[ ] Deprecated patterns absent — no known deprecated APIs used?
[ ] Async pattern matches project style — callbacks vs promises vs async/await?
[ ] Module system matches — CommonJS vs ESM consistent with project?
[ ] Types added (if TypeScript project) — proper annotations included?
```

---

## Import Verification Steps

### For npm/Node.js Packages

**Step 1** — Find the package's current exports:

```bash
# Local: if already installed
node -e "console.log(Object.keys(require('{package}')))"

# Or inspect source: browse to main entry point on GitHub
# github.com/{owner}/{repo}/blob/main/{main-entry}.js
# Look for: module.exports = { ... } or export { ... }
```

**Step 2** — Cross-reference against the example's imports:

```javascript
// Example imports:
import { rateLimit, MemoryStore } from 'express-rate-limit'

// Verify: does 'express-rate-limit' export both 'rateLimit' and 'MemoryStore'?
// Check: npmjs.com/package/express-rate-limit → Repository → src/lib/index.ts
```

**Step 3** — Check for renamed exports in changelog:

```
Search: "express-rate-limit changelog renamed" OR site:github.com/express-rate-limit CHANGELOG.md
Look for: "renamed X to Y", "X is now Y", "breaking: removed X"
```

### For Python Packages

**Step 1** — Find current module structure:

```python
# If installed:
import {package}; print(dir({package}))

# Or check PyPI → browse GitHub → look at __init__.py
```

**Step 2** — Verify specific imports:

```python
# Example import:
from fastapi import Depends, HTTPException, status

# Verify: does fastapi export Depends, HTTPException, status?
# Check: github.com/tiangolo/fastapi/blob/master/fastapi/__init__.py
```

### For Rust Crates

```bash
# Check docs.rs for current public API:
# docs.rs/{crate}/{version}/{crate}/index.html

# Example: docs.rs/tokio/1.0.0/tokio/index.html
# Verify struct/trait/fn names match
```

---

## API Surface Comparison

### Method Signature Verification

For critical methods, verify signature hasn't changed:

```javascript
// Found example uses:
redis.set(key, value, 'EX', ttl)

// Current Redis client (ioredis v5) uses:
redis.set(key, value, 'EX', ttl)  // same — confirmed

// But node-redis v4 changed to:
redis.set(key, value, { EX: ttl })  // different options format
```

**How to verify:**

1. Go to official docs for the specific version
2. Find the method's signature in the API reference
3. Compare parameter types, order, and option shapes

### Deprecation Pattern Detection

These patterns indicate deprecated APIs that should be replaced:

```javascript
// Node.js deprecations
new Buffer(...)            // Deprecated — use Buffer.from() or Buffer.alloc()
require('url').parse()     // Prefer: new URL()
fs.exists()               // Deprecated — use fs.access()
process.binding()         // Internal, avoid

// Express deprecations
res.send(404)             // Old — use res.status(404).send()
app.configure()           // Removed in Express 4

// React deprecations
componentWillMount()      // Deprecated — use useEffect()
React.createClass()       // Removed — use class components or hooks
findDOMNode()             // Deprecated — use refs
```

```python
# Python deprecations
asyncio.coroutine          # Deprecated since 3.8, removed in 3.11
asyncio.ensure_future()    # Prefer asyncio.create_task()
collections.Mapping        # Use collections.abc.Mapping (since 3.3, removed 3.10)
typing.List, typing.Dict   # Use list, dict directly (Python 3.9+)
```

---

## Module System Compatibility

### Detecting the Project's Module System

```bash
# Check for ESM signals:
grep -r '"type": "module"' package.json  # ESM if present
grep -r 'import .* from' src/ --include="*.js" | head -3  # ESM syntax

# Check for CJS signals:
grep -r 'require(' src/ --include="*.js" | head -3  # CJS syntax

# TypeScript projects: check tsconfig.json
grep '"module"' tsconfig.json  # "ESNext" or "ES2022" = ESM target
```

### Adaptation Table

| Found Example Style | Project Style | Adaptation Needed |
|--------------------|---------------|------------------|
| `require()` | CJS | None |
| `require()` | ESM | Convert to `import` |
| `import` | ESM | None |
| `import` | CJS (older Node) | Convert to `require()` OR add `"type":"module"` |
| `import` | TypeScript | Usually none, check `moduleResolution` |

---

## Type Annotation Addition Patterns

When adapting JS examples for TypeScript projects:

### Function Parameters

```typescript
// Before (JS):
async function getUser(id, db) {
  return db.find(id)
}

// After (TS):
import type { Database } from './types'
import type { User } from './models'

async function getUser(id: number, db: Database): Promise<User | null> {
  return db.find(id)
}
```

### Event Handlers

```typescript
// Before (JS):
document.addEventListener('click', (e) => console.log(e.target))

// After (TS):
document.addEventListener('click', (e: MouseEvent) => {
  console.log((e.target as HTMLElement).id)
})
```

### Generic Collections

```typescript
// Before (JS):
const cache = new Map()

// After (TS):
const cache = new Map<string, UserData>()
```

### Checking for Built-in Types vs @types/ Packages

```bash
# Check if library has built-in TypeScript types:
cat node_modules/{library}/package.json | grep '"types"\|"typings"'
# If present: built-in types, no @types/ needed

# If not present, check @types/:
npm info @types/{library} version
# If found: install with: npm install --save-dev @types/{library}
```

---

## Common Adaptation Anti-Patterns

Avoid these common mistakes when adapting examples:

### Don't Hardcode What Should Be Config

```javascript
// Bad adaptation:
const redis = new Redis({ host: 'localhost', port: 6379 })

// Good adaptation:
const redis = new Redis({
  host: process.env.REDIS_HOST ?? 'localhost',
  port: parseInt(process.env.REDIS_PORT ?? '6379', 10),
})
```

### Don't Swallow Errors

```javascript
// Bad adaptation (from incomplete example):
try {
  const result = await doThing()
} catch (err) {
  // TODO: handle error
}

// Good adaptation:
try {
  const result = await doThing()
} catch (err) {
  logger.error('doThing failed', { err })
  throw err  // or handle specifically
}
```

### Don't Mix Async Patterns

```javascript
// Bad: mixing async/await with .then() chains
const result = await doThing().then(r => r.data)

// Good: pick one style
const result = (await doThing()).data
```
