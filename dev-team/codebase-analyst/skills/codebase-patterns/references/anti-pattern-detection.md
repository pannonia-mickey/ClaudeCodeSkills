# Anti-Pattern Detection

## God Classes / God Modules

Large files or classes with too many responsibilities violate the Single Responsibility Principle.

### Detection Heuristics

```bash
# Files over 500 lines (potential god classes)
find . -type f \( -name '*.py' -o -name '*.ts' -o -name '*.js' -o -name '*.go' -o -name '*.rs' \) \
  -not -path '*/node_modules/*' -not -path '*/vendor/*' \
  -exec wc -l {} + | sort -rn | awk '$1 > 500 {print}' | head -15

# Classes with many methods (Python)
for f in $(find . -name '*.py' -not -path '*/venv/*'); do
  methods=$(grep -c "    def " "$f" 2>/dev/null)
  [ "$methods" -gt 15 ] && echo "$methods methods: $f"
done | sort -rn | head -10

# Classes with many methods (TypeScript)
for f in $(find . -name '*.ts' -not -path '*/node_modules/*'); do
  methods=$(grep -cE "^\s+(async )?(public |private |protected )?[a-zA-Z].*\(" "$f" 2>/dev/null)
  [ "$methods" -gt 15 ] && echo "$methods methods: $f"
done | sort -rn | head -10

# Files with many imports (high coupling)
for f in $(find . -name '*.ts' -not -path '*/node_modules/*'); do
  imports=$(grep -c "^import " "$f" 2>/dev/null)
  [ "$imports" -gt 15 ] && echo "$imports imports: $f"
done | sort -rn | head -10
```

### Remediation
- Extract cohesive groups of methods into separate services/classes
- Apply Single Responsibility Principle: each class should have one reason to change
- Use composition to combine smaller, focused classes

## Circular Dependencies

Modules that import each other create tight coupling and complicate reasoning.

### Detection Heuristics

```bash
# Python: find modules that import each other
# Build import pairs and check for cycles
for f in $(find src/ -name '*.py' 2>/dev/null); do
  module=$(echo "$f" | sed 's/src\///' | sed 's/\.py//' | sed 's/\//__/g' | sed 's/__init//')
  imports=$(grep "^from \.\|^from src\." "$f" 2>/dev/null | sed 's/from //;s/ import.*//')
  for imp in $imports; do
    echo "$module -> $imp"
  done
done | sort > /tmp/deps.txt

# Check for A->B and B->A
awk -F' -> ' '{
  key = ($1 < $2) ? $1" "$2 : $2" "$1
  if (key in seen) print "CIRCULAR: "$1" <-> "$2
  seen[key] = 1
}' /tmp/deps.txt

# TypeScript: check for circular imports
# If using ESLint, check for import/no-cycle rule
grep -rn "no-cycle\|import/no-cycle" .eslintrc* eslint.config.* 2>/dev/null
```

### Remediation
- Extract shared types/interfaces into a separate module
- Use dependency inversion (depend on abstractions, not implementations)
- Introduce an event system for loose coupling
- Restructure module boundaries to eliminate cycles

## Dead Code

Unreferenced exports, unused imports, and abandoned functions waste cognitive load.

### Detection Heuristics

```bash
# Unused exports (TypeScript) — find exports not imported elsewhere
for f in $(find src/ -name '*.ts' -not -name '*.test.*' -not -name '*.spec.*' 2>/dev/null); do
  exports=$(grep -oP 'export (function|class|const|type|interface|enum) \K[a-zA-Z]+' "$f" 2>/dev/null)
  for exp in $exports; do
    count=$(grep -rn "$exp" --include="*.ts" | grep -v "$f" | wc -l)
    [ "$count" -eq 0 ] && echo "UNUSED EXPORT: $exp in $f"
  done
done | head -20

# Unused imports (crude heuristic — check if imported name appears in file body)
# Better: rely on IDE/linter (TypeScript: noUnusedLocals)
grep -rn "\"noUnusedLocals\"\|\"noUnusedParameters\"" tsconfig.json 2>/dev/null

# Python: unused imports
grep -rn "^import \|^from .* import " --include="*.py" | head -30
# Better: use flake8 F401 or pylint unused-import

# Files not imported anywhere
for f in $(find src/ -name '*.ts' -not -name 'index.ts' -not -name '*.test.*' 2>/dev/null); do
  base=$(basename "$f" .ts)
  count=$(grep -rn "from.*$base\|import.*$base" --include="*.ts" | grep -v "$f" | wc -l)
  [ "$count" -eq 0 ] && echo "ORPHAN FILE: $f"
done | head -20
```

### Remediation
- Delete unused exports, imports, and files
- Enable linter rules: `noUnusedLocals`, `noUnusedParameters`, `F401`
- Run periodic dead code analysis as part of CI

## Copy-Paste Code (Duplication)

Duplicated logic creates maintenance burden — changes must be applied in multiple places.

### Detection Heuristics

```bash
# Find files with suspiciously similar structure
# Compare function signatures across files
grep -rn "^export async function\|^def \|^func " \
  --include="*.ts" --include="*.py" --include="*.go" \
  | sed 's/.*function \|.*def \|.*func //' | sed 's/(.*//' \
  | sort | uniq -d | head -10
# Duplicate function names may indicate copy-paste

# Find nearly identical files by line count
find . -name '*.ts' -not -path '*/node_modules/*' -exec wc -l {} + \
  | sort -n | awk 'prev && $1 == prev_count {print prev; print $0} {prev=$0; prev_count=$1}' \
  | head -20

# Find similar error handling blocks
grep -rn "try {" --include="*.ts" -A 10 | head -50
# Look for repeated catch patterns manually

# Repeated SQL queries or API calls
grep -rn "SELECT \|INSERT \|UPDATE \|DELETE " --include="*.ts" --include="*.py" \
  | sed 's/.*SELECT/SELECT/;s/.*INSERT/INSERT/;s/.*UPDATE/UPDATE/;s/.*DELETE/DELETE/' \
  | sort | uniq -c | sort -rn | head -10
```

### Remediation
- Extract shared logic into utility functions or services
- Create shared base classes or mixins for common behavior
- Use generics/templates for type-safe reuse

## Inappropriate Coupling

Modules reaching into other modules' internals instead of using public APIs.

### Detection Heuristics

```bash
# Cross-layer imports (presentation importing data layer directly)
# Controllers should not import repositories
grep -rn "from.*repository\|import.*Repository" --include="*.ts" --include="*.py" \
  | grep -iE "controller|route|handler|view" | head -10

# Deep relative imports (reaching far up the tree)
grep -rn "from '\.\./\.\./\.\./\|from \"\.\./\.\./\.\./" --include="*.ts" | head -10
grep -rn "from \.\.\.\." --include="*.py" | head -10

# Importing private/internal modules from outside
grep -rn "from.*_internal\|from.*private\|import.*_private" \
  --include="*.ts" --include="*.py" | head -10

# Go: importing internal packages from outside
grep -rn "\".*internal/" --include="*.go" | head -10
```

### Remediation
- Enforce layer boundaries through linter rules or module visibility
- Use dependency injection to provide implementations
- Define clear public APIs for each module (barrel exports, `__all__`)

## Primitive Obsession

Using primitive types (strings, numbers) where domain-specific types would be clearer.

### Detection Heuristics

```bash
# Functions with many string/number parameters
grep -rn "function.*string, string, string\|def.*str, str, str" \
  --include="*.ts" --include="*.py" | head -10

# ID parameters as plain strings (should be branded/newtype)
grep -rn "userId: string\|orderId: string\|user_id: str\|order_id: str" \
  --include="*.ts" --include="*.py" | head -10

# Money as plain numbers (should be a value object)
grep -rn "price: number\|amount: number\|total: number\|cost: number" \
  --include="*.ts" | head -10

# Status as plain strings (should be enum)
grep -rn "status: string\|status: str\|status === ['\"]" \
  --include="*.ts" --include="*.py" | head -10
```

### Remediation
- Create branded types or newtypes for IDs: `type UserId = string & { __brand: 'UserId' }`
- Use enums for finite sets: `enum OrderStatus { Pending, Shipped, Delivered }`
- Create value objects for monetary amounts, dates, emails
- Use Pydantic/Zod for validated domain types

## Long Parameter Lists

Functions with many parameters are hard to call correctly and maintain.

### Detection Heuristics

```bash
# Functions with 5+ parameters (TypeScript)
grep -rn "function.*,.*,.*,.*,.*," --include="*.ts" | head -10

# Functions with 5+ parameters (Python)
grep -rn "def .*,.*,.*,.*,.*," --include="*.py" | head -10

# Constructor injection with many dependencies
grep -rn "constructor(" --include="*.ts" -A 10 | grep -B1 "private\|readonly" | head -20
```

### Remediation
- Group related parameters into an options/config object
- Use the Builder pattern for complex construction
- Split the function into smaller, focused functions
- Use dependency injection containers to manage dependencies
