---
name: Codebase Patterns
description: This skill should be used when the user asks about "coding conventions", "naming conventions", "code patterns", "design patterns", "error handling patterns", "logging patterns", "code style", "convention discovery", "anti-patterns", or "code organization". It covers pattern recognition, convention extraction, and anti-pattern detection in existing codebases.
---

# Pattern Recognition & Convention Discovery

## Naming Convention Detection

Extract naming patterns from the codebase to document the project's style guide.

### File Naming

```bash
# Detect file naming convention
find src/ -name '*.ts' -not -path '*/node_modules/*' | xargs -I{} basename {} \
  | head -30 | sort

# Check: kebab-case, camelCase, PascalCase, or snake_case
# kebab-case: user-service.ts, order-controller.ts
# camelCase: userService.ts, orderController.ts
# PascalCase: UserService.ts, OrderController.ts
# snake_case: user_service.py, order_controller.py
```

### Symbol Naming

```bash
# Class names (expect PascalCase in most languages)
grep -rn "^class \|^export class " --include="*.ts" --include="*.py" \
  | sed 's/.*class //' | sed 's/[({: ].*//' | head -20

# Function names
grep -rn "^function \|^export function \|^def \|^async def " \
  --include="*.ts" --include="*.py" | sed 's/.*function \|.*def //' \
  | sed 's/(.*//' | head -20

# Constant names (expect UPPER_SNAKE_CASE)
grep -rn "^const [A-Z_]*\s*=\|^[A-Z_]* = " --include="*.ts" --include="*.py" | head -10

# Interface/type naming (I-prefix vs no prefix)
grep -rn "^interface \|^export interface \|^type " --include="*.ts" \
  | sed 's/.*interface \|.*type //' | sed 's/[{ =<].*//' | head -20
```

## Error Handling Pattern Identification

### Pattern Detection

```bash
# Custom error classes
grep -rn "class.*Error.*extends\|class.*Exception" --include="*.ts" --include="*.py" | head -10

# Result/Either type pattern
grep -rn "Result<\|Either<\|Ok(\|Err(" --include="*.ts" --include="*.rs" | head -10

# Try-catch density
total_lines=$(find . -name '*.ts' -not -path '*/node_modules/*' -exec cat {} + | wc -l)
try_count=$(grep -rn "try {" --include="*.ts" | wc -l)
echo "Try-catch ratio: $try_count catches in $total_lines lines"

# HTTP error handling
grep -rn "HttpException\|HttpError\|status(4\|status(5\|raise.*Http" \
  --include="*.ts" --include="*.py" | head -10

# Centralized error handler
grep -rn "errorHandler\|error_handler\|exception_handler\|@ExceptionFilter" \
  --include="*.ts" --include="*.py" | head -5
```

### Common Error Patterns

| Pattern | Indicator | Languages |
|---------|-----------|-----------|
| Custom error hierarchy | `class AppError extends Error` | TS, Python, Java |
| Result/Either type | `Result<T, E>`, `Ok()`, `Err()` | Rust, TS (fp-ts) |
| Error codes | `ErrorCode.NOT_FOUND` | Any |
| HTTP error mapping | `HttpException`, `HTTPException` | Express, FastAPI |
| Global error handler | `app.use(errorHandler)` | Express, NestJS |

## Logging Pattern Analysis

```bash
# Detect logging library
grep -rn "winston\|pino\|bunyan\|morgan" --include="*.ts" --include="*.js" | head -5
grep -rn "import logging\|structlog\|loguru" --include="*.py" | head -5
grep -rn "slog\|logrus\|zap\." --include="*.go" | head -5

# Logging style: structured vs unstructured
grep -rn "logger\.\(info\|error\|warn\|debug\)" --include="*.ts" --include="*.py" | head -10

# Log level distribution
for level in debug info warn error; do
  count=$(grep -rn "logger\.$level\|log\.$level\|logging\.$level" \
    --include="*.ts" --include="*.py" | wc -l)
  echo "$level: $count occurrences"
done
```

## Configuration Management Patterns

```bash
# Environment variable usage
grep -rn "process\.env\.\|os\.environ\|os\.getenv\|env\." \
  --include="*.ts" --include="*.py" --include="*.go" | head -20

# Configuration file patterns
find . -maxdepth 2 -name 'config.*' -o -name 'settings.*' -o -name '*.config.*'

# Config validation (Pydantic, Zod, Joi)
grep -rn "BaseSettings\|z\.object\|Joi\.object\|@IsString\|@IsNumber" \
  --include="*.ts" --include="*.py" | head -10

# Feature flags
grep -rn "feature.*flag\|FEATURE_\|isEnabled\|unleash\|launchdarkly" \
  --include="*.ts" --include="*.py" | head -10
```

## Import Organization Patterns

```bash
# TypeScript import ordering
head -30 src/services/*.ts 2>/dev/null | grep "^import"

# Typical order:
# 1. Node built-ins (fs, path, crypto)
# 2. External packages (express, lodash)
# 3. Internal absolute imports (@/services, ~/utils)
# 4. Relative imports (./helpers, ../models)

# Python import ordering
head -30 src/services/*.py 2>/dev/null | grep "^import\|^from"

# Typical order:
# 1. Standard library (os, sys, typing)
# 2. Third-party (fastapi, sqlalchemy)
# 3. Local application (from . import, from ..models)

# Path alias detection
grep -rn "\"@/\|'@/\|\"~/\|'~/" --include="*.ts" | head -5
grep "paths\|alias" tsconfig.json 2>/dev/null | head -10
```

## Design Pattern Recognition

```bash
# Repository pattern
grep -rn "Repository\|repository" --include="*.ts" --include="*.py" | head -10

# Factory pattern
grep -rn "Factory\|factory\|create.*Instance\|build.*From" \
  --include="*.ts" --include="*.py" | head -10

# Observer/Event pattern
grep -rn "EventEmitter\|addEventListener\|subscribe\|Observer\|on(" \
  --include="*.ts" --include="*.py" | head -10

# Singleton pattern
grep -rn "getInstance\|_instance\|@singleton" --include="*.ts" --include="*.py" | head -10

# Strategy pattern
grep -rn "Strategy\|strategy\|Policy\|policy" --include="*.ts" --include="*.py" | head -10

# Decorator pattern
grep -rn "^@\|@Inject\|@Injectable\|@Module\|@Component" --include="*.ts" --include="*.py" | head -10

# Dependency Injection
grep -rn "inject\|@Inject\|@Injectable\|Container\|provide\|resolve" \
  --include="*.ts" --include="*.py" | head -10
```

## Code Organization Conventions

### Feature-Based vs Layer-Based

```bash
# Feature-based (files grouped by domain feature)
# src/users/user.controller.ts, src/users/user.service.ts
find . -maxdepth 2 -type d | grep -E 'src/[a-z]+s?$' | head -10

# Layer-based (files grouped by technical concern)
# src/controllers/user.controller.ts, src/services/user.service.ts
find . -maxdepth 2 -type d | grep -iE 'controller|service|repository|model' | head -10
```

## References

- [Convention Catalog](references/convention-catalog.md) — Comprehensive naming conventions, import ordering, and formatting standards across Python, JavaScript/TypeScript, Go, and Rust with detection commands.
- [Anti-Pattern Detection](references/anti-pattern-detection.md) — Detection heuristics for god classes, circular dependencies, dead code, copy-paste duplication, and inappropriate coupling.
