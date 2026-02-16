# Convention Catalog

## Python Conventions

### PEP 8 Compliance Detection

```bash
# File naming: snake_case.py
find . -name '*.py' -not -path '*/venv/*' | xargs -I{} basename {} \
  | grep -vE '^[a-z_]+\.py$' | head -10
# Non-empty output = files violating snake_case

# Class naming: PascalCase
grep -rn "^class " --include="*.py" | sed 's/.*class //' | sed 's/[(:].*//' \
  | grep -vE '^[A-Z][a-zA-Z0-9]*$' | head -10

# Function naming: snake_case
grep -rn "^def \|^    def " --include="*.py" | sed 's/.*def //' | sed 's/(.*//' \
  | grep -vE '^_?_?[a-z_][a-z0-9_]*_?_?$' | head -10

# Constant naming: UPPER_SNAKE_CASE
grep -rn "^[A-Z_][A-Z_0-9]* = " --include="*.py" | head -10
```

### Import Ordering

```python
# Standard PEP 8 / isort ordering:
import os                           # 1. Standard library
import sys
from pathlib import Path

import httpx                        # 2. Third-party packages
from fastapi import FastAPI
from pydantic import BaseModel

from app.config import settings     # 3. Local application
from app.models import User
from .utils import helper           # 4. Relative imports
```

```bash
# Detect isort configuration
find . -name '.isort.cfg' -o -name 'setup.cfg' | xargs grep -l "isort" 2>/dev/null
grep "isort\|import-sort" pyproject.toml 2>/dev/null
```

### Type Hint Usage

```bash
# Type hint coverage (Python 3.10+ syntax)
grep -rn "def " --include="*.py" | head -30 | grep -c " -> "
# Compare against total function count to estimate coverage

# Modern union syntax (3.10+)
grep -rn "int | str\|None |" --include="*.py" | head -5

# Legacy Optional usage
grep -rn "Optional\[" --include="*.py" | head -5
```

### Docstring Style

```bash
# Google-style docstrings
grep -A5 '"""' --include="*.py" -rn | grep -E "Args:|Returns:|Raises:" | head -5

# NumPy-style docstrings
grep -A5 '"""' --include="*.py" -rn | grep -E "Parameters|Returns|Raises" | head -5

# Sphinx-style docstrings
grep -rn ":param \|:returns:\|:raises:" --include="*.py" | head -5
```

## JavaScript / TypeScript Conventions

### ESLint Configuration Detection

```bash
# Find ESLint config
find . -maxdepth 2 -name '.eslintrc*' -o -name 'eslint.config.*'

# Prettier config
find . -maxdepth 2 -name '.prettierrc*' -o -name 'prettier.config.*'

# Check for specific rule sets
grep -l "airbnb\|standard\|recommended" .eslintrc* eslint.config.* 2>/dev/null
```

### Naming Conventions

```
Files:       kebab-case.ts    or    camelCase.ts
Components:  PascalCase.tsx   (React convention)
Tests:       *.test.ts  or  *.spec.ts
Types:       PascalCase (no I- prefix in modern TS)
Interfaces:  PascalCase (legacy: IUserService, modern: UserService)
Enums:       PascalCase with PascalCase members
Constants:   UPPER_SNAKE_CASE or camelCase
```

```bash
# Detect React component file naming
find . -name '*.tsx' -not -path '*/node_modules/*' | xargs -I{} basename {} | head -20

# Check interface naming (I-prefix or not)
grep -rn "^export interface I[A-Z]" --include="*.ts" | wc -l    # I-prefix count
grep -rn "^export interface [^I]" --include="*.ts" | wc -l       # Non-prefix count
```

### Barrel Exports

```bash
# Check for barrel export pattern (re-exports from index.ts)
find . -name 'index.ts' -not -path '*/node_modules/*' -exec grep -l "export.*from" {} \;

# Example barrel:
# export { UserService } from './user.service';
# export { UserController } from './user.controller';
# export type { User } from './user.types';
```

### Module System

```bash
# CommonJS vs ESM detection
grep -rn "require(" --include="*.ts" --include="*.js" | wc -l    # CommonJS
grep -rn "^import " --include="*.ts" --include="*.js" | wc -l    # ESM

# package.json type field
grep '"type"' package.json 2>/dev/null
# "type": "module" = ESM, "type": "commonjs" or absent = CJS
```

## Go Conventions

### Go Formatting (Enforced by gofmt)

```bash
# Check if gofmt is enforced in CI
grep -rn "gofmt\|goimports\|golangci-lint" .github/ Makefile 2>/dev/null

# Package naming (single lowercase word)
find . -name '*.go' -exec dirname {} \; | sort -u | xargs -I{} basename {}

# Exported vs unexported (capitalization)
grep -rn "^func [A-Z]" --include="*.go" | wc -l    # Exported
grep -rn "^func [a-z]" --include="*.go" | wc -l    # Unexported
```

### Error Handling (Go Idiom)

```bash
# Standard Go error pattern: if err != nil
grep -rn "if err != nil" --include="*.go" | wc -l

# Error wrapping (Go 1.13+)
grep -rn "fmt\.Errorf.*%w\|errors\.Wrap\|errors\.New" --include="*.go" | head -10

# Sentinel errors
grep -rn "var Err[A-Z].*= errors\.\|var Err[A-Z].*= fmt\." --include="*.go" | head -10
```

### Project Layout

```bash
# Standard Go project layout indicators
ls cmd/ 2>/dev/null && echo "cmd/ found (CLI entry points)"
ls internal/ 2>/dev/null && echo "internal/ found (private packages)"
ls pkg/ 2>/dev/null && echo "pkg/ found (public packages)"
ls api/ 2>/dev/null && echo "api/ found (API definitions)"
```

## Rust Conventions

### Clippy and Formatting

```bash
# Check for clippy in CI
grep -rn "clippy\|cargo fmt" .github/ Makefile 2>/dev/null

# Deny warnings configuration
grep -rn "#!\[deny\|#!\[warn\|#!\[allow" --include="*.rs" | head -10

# Clippy configuration
find . -name 'clippy.toml' -o -name '.clippy.toml'
```

### Module Organization

```bash
# Module tree
find . -name 'mod.rs' -o -name 'lib.rs' -o -name 'main.rs' | sort

# Public vs private modules
grep -rn "^pub mod\|^mod " --include="*.rs" | head -20

# Re-exports
grep -rn "^pub use " --include="*.rs" | head -10
```

## General Cross-Language Patterns

### Configuration File Conventions

| Convention | Detection |
|-----------|-----------|
| `.env` for secrets | `find . -name '.env*'` |
| YAML for config | `find . -name '*.yml' -o -name '*.yaml'` |
| TOML for Rust/Python | `find . -name '*.toml'` |
| JSON for Node.js | `find . -name '*.config.json'` |

### Directory Organization Patterns

```bash
# Standard directories
for dir in src lib test tests spec docs config scripts bin assets public static; do
  [ -d "$dir" ] && echo "FOUND: $dir/"
done

# Test colocation (tests next to source) vs separation (tests/ directory)
find . -name '*.test.*' -o -name '*.spec.*' | head -5 | grep "src/" \
  && echo "Colocated tests" || echo "Separated tests"
```

### Git Conventions

```bash
# Commit message style
git log --oneline -20

# Conventional commits detection
git log --oneline -20 | grep -cE "^[a-f0-9]+ (feat|fix|chore|docs|style|refactor|test|perf|ci)(\(.*\))?:"

# Branch naming convention
git branch -a | head -20
```
