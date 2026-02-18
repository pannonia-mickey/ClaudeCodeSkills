---
name: Research Code Examples
triggers:
  - "show me an example"
  - "how do I implement"
  - "code for"
  - "working example"
  - "example of"
  - "sample code"
  - "code snippet"
description: |
  This skill should be used when the research-expert needs to find, validate, and adapt working code examples for a specific technology, library, or API pattern. It covers where to locate examples in order of reliability, how to verify examples against the target version, and how to adapt generic examples to the user's specific technology stack.
---

# Research Code Examples: Finding, Validating, and Adapting Working Code

## Where to Find Examples

Search in priority order. Stop when you find an example that is version-correct and runnable.

### Priority 1 — Official Documentation Examples

The most reliable examples are in the official documentation, in this order:

1. **Quickstart / Getting Started guide** — minimal working example, maintained by the team
2. **API reference examples** — per-method examples, often the most accurate
3. **Official GitHub `examples/` directory** — runnable project examples, usually tested in CI
4. **Library test suite** (`test/`, `spec/`, `__tests__/`) — real usage, guaranteed to work with that version

Finding the official `examples/` directory:
```
Search: "github.com/{owner}/{repo}/tree/main/examples"
Or navigate: github.com/{owner}/{repo} → browse to examples/ folder
```

### Priority 2 — Package Registry Pages

Package registry pages often embed the most commonly-needed usage pattern:

- **npm**: npmjs.com/{package-name} — README with installation and basic usage
- **PyPI**: pypi.org/project/{package-name} — project description with quickstart
- **crates.io**: crates.io/crates/{crate} — links to docs.rs which has API examples

### Priority 3 — GitHub Code Search

For patterns not in official docs, search real-world usage:

```
# Find usage in open-source projects
{function-name} language:{language} in:file filename:*.{ext}

# Examples:
"rateLimit(" language:javascript in:file filename:*.js
"@pytest.mark.asyncio" language:python in:file filename:test_*.py
```

GitHub code search: github.com/search?type=code&q={query}

### Priority 4 — Stack Overflow (Verified Answers Only)

Criteria for a trustworthy SO example:

- ✓ Green checkmark (accepted by question asker)
- ✓ 20+ upvotes (community endorsement)
- ✓ Comments don't say "this is outdated" or "this won't work in v{current}"
- ✓ Posted within last 2 years for fast-moving ecosystems (React, Node.js, Python packaging)

URL for filtered search: `https://stackoverflow.com/search?q={query}&tab=votes`

### Priority 5 — Library's GitHub Issues

Sometimes the only working example for an edge case is a maintainer's reply in an issue:

```
# Search GitHub issues for examples
github.com/{owner}/{repo}/issues?q={feature}+example

# Or use web search
site:github.com/{owner}/{repo}/issues "{feature}" example
```

---

## Version-Pinned Searching

Always include version information in queries when the API has changed between versions.

### How to Identify the Version in Use

```bash
# npm
cat package.json | grep '"{library}"'
# or: npm list {library}

# Python
pip show {library}
# or: cat requirements.txt | grep {library}

# Rust
cat Cargo.toml | grep '{crate}'

# Go
cat go.mod | grep '{module}'
```

### Formulating Version-Specific Queries

```
# Include major version in query
express 4 middleware example
react 18 suspense data fetching
python 3.11 asyncio gather
axios 1.x interceptors

# Use version-specific docs URLs
docs.djangoproject.com/en/4.2/  (not /en/latest/ — stable URL but version-locked)
react.dev (new docs, assumes React 18+)
```

### Checking for API Changes Between Versions

When the example's version differs from the project's version:

1. Find the CHANGELOG:
   ```
   github.com/{owner}/{repo}/blob/main/CHANGELOG.md
   github.com/{owner}/{repo}/releases
   ```

2. Search for the function/class name in the changelog:
   ```
   # In browser: Ctrl+F for the method name in CHANGELOG.md
   # Or: search "{library} {function} deprecated"
   ```

3. Check the migration guide if major version change:
   ```
   Search: "{library} migrate from v{old} to v{new}"
   ```

---

## Example Validation

Before presenting any code example, run it through this validation sequence.

### Step 1 — Import Check

Verify every import still exists in the current version:

```
For each import in the example:
  Search: "{library} {exported-name} API reference"
  Verify: the exported name appears in current API docs/source
  If not found: search for renamed equivalent or note removal
```

**Common import changes to watch for:**

```javascript
// Old (pre-ESM):
const express = require('express')
// New (ESM):
import express from 'express'
// Both valid but check project's module system

// React pre-17:
import React from 'react' // required for JSX
// React 17+:
// Not required for JSX, but harmless

// Axios v0.x:
axios.get(url).then(res => res.data)
// Axios v1.x:
// Same API, but different error structure
```

### Step 2 — Method Signature Check

Verify that method signatures match (parameter order, required vs optional):

```
Search: "{library} {method-name} signature" site:{official-docs}
Or: Browse github.com/{owner}/{repo}/blob/main/src/{relevant-file}
```

### Step 3 — Async Pattern Check

Verify whether the example uses the right async pattern for the version:

```javascript
// Callback-style (older):
fs.readFile(path, 'utf8', (err, data) => { ... })

// Promise-style (Node 10+):
const data = await fs.promises.readFile(path, 'utf8')

// Check which pattern the project uses:
grep -r "fs.promises\|require('fs/promises')" src/
```

### Step 4 — Configuration Option Check

If the example includes a configuration object, verify each option still exists:

```
Search: "{library} configuration options" OR "{library} options reference"
Confirm each key in the config object is documented in current version
```

---

## Context Adaptation

Generic examples need adaptation to fit the user's specific stack. Apply these transformations:

### JavaScript Module System Adaptation

```javascript
// If project uses CommonJS (has require() in existing files):
const rateLimit = require('express-rate-limit')

// If project uses ESM (has import/export in existing files, or "type": "module" in package.json):
import rateLimit from 'express-rate-limit'
```

### TypeScript Adaptation

```typescript
// Add type annotations to generic JS examples:
// Before (JS):
const handler = (req, res) => { ... }

// After (TS):
import { Request, Response } from 'express'
const handler = (req: Request, res: Response): void => { ... }

// Check if @types package needed:
// npm install --save-dev @types/{library}
// (Not needed for libraries with built-in types — check package.json: "types" field)
```

### Async/Await vs Promise Chain Adaptation

```javascript
// Promise chain (in example):
doThing()
  .then(result => process(result))
  .catch(err => handleError(err))

// Async/await equivalent:
try {
  const result = await doThing()
  process(result)
} catch (err) {
  handleError(err)
}
```

### Framework-Specific Adaptation Patterns

```python
# Adapting a generic Python example to FastAPI:
# Before (generic):
def get_user(user_id):
    return db.query(user_id)

# After (FastAPI):
from fastapi import Depends
async def get_user(user_id: int, db: Session = Depends(get_db)):
    return db.query(User).filter(User.id == user_id).first()
```

---

## Citation Format

Always cite examples with enough context to re-verify:

```markdown
**Source**: [{Library} Official Docs — {Section}]({URL})
**Version**: {library} v{X.Y} (project uses v{X.Y})
**Adapted**: {describe any changes made to the original example}
```

For Stack Overflow:
```markdown
**Source**: [Stack Overflow — {Question Title}]({URL})
**Answer by**: {username} ({reputation} rep, accepted answer, {upvotes} upvotes)
**Verified**: {date} — comments confirm it works with v{X.Y}
```

---

## References

For import verification steps, API surface comparison, and module system adaptation patterns, see [code-validation.md](references/code-validation.md).

For a curated guide to finding examples by ecosystem (npm, PyPI, Rust, APIs), GitHub search patterns, and documentation site conventions, see [example-sources.md](references/example-sources.md).
