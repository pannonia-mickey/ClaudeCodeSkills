# Analysis Methodology

## Phase 1: Reconnaissance — Deep Dive

### Directory Structure Mapping

```bash
# Full tree with depth limit (excludes node_modules, .git, vendor, __pycache__)
find . -maxdepth 4 -type d \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/vendor/*' \
  -not -path '*/__pycache__/*' \
  -not -path '*/dist/*' \
  -not -path '*/build/*' \
  | sort

# File count per directory (identify heavy modules)
find . -maxdepth 2 -type d | while read dir; do
  count=$(find "$dir" -maxdepth 1 -type f | wc -l)
  echo "$count $dir"
done | sort -rn | head -20
```

### Build System Detection

| Build Tool | Detection | Ecosystem |
|-----------|-----------|-----------|
| npm/pnpm/yarn | `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock` | JavaScript |
| pip/poetry/uv | `requirements.txt`, `poetry.lock`, `uv.lock` | Python |
| cargo | `Cargo.lock` | Rust |
| go modules | `go.sum` | Go |
| maven/gradle | `pom.xml`, `build.gradle` | Java |
| dotnet | `*.sln`, `*.csproj` | .NET |
| cmake/make | `CMakeLists.txt`, `Makefile` | C/C++ |

```bash
# Detect build scripts
grep -l '"scripts"' package.json 2>/dev/null && echo "npm scripts found"
ls Makefile 2>/dev/null && echo "Makefile found"
ls justfile 2>/dev/null && echo "justfile found"
ls Taskfile.yml 2>/dev/null && echo "Taskfile found"

# Read build/start commands
jq '.scripts' package.json 2>/dev/null
grep -E '^[a-zA-Z].*:' Makefile 2>/dev/null | head -20
```

### Environment Configuration

```bash
# Environment files
find . -maxdepth 2 -name '.env*' -o -name '*.env' | head -10

# Configuration directories
find . -maxdepth 2 -type d -name 'config' -o -name 'settings' -o -name 'conf'

# Docker configuration
find . -maxdepth 2 -name 'Dockerfile*' -o -name 'docker-compose*' -o -name '.dockerignore'

# CI/CD configuration
find . -maxdepth 3 -name '*.yml' -o -name '*.yaml' \
  | grep -E '(github|gitlab|jenkins|circle|travis|azure)'
```

## Phase 2: Architecture Extraction — Deep Dive

### Module Boundary Detection

```bash
# Python package detection (directories with __init__.py)
find . -name '__init__.py' -exec dirname {} \; | sort

# JavaScript/TypeScript module detection (index files)
find . -name 'index.ts' -o -name 'index.js' | sed 's/\/index\..*//' | sort -u

# Go package detection
find . -name '*.go' -exec dirname {} \; | sort -u

# Rust module detection
find . -name 'mod.rs' -exec dirname {} \; | sort -u
```

### Layer Identification Heuristics

Look for these directory patterns to identify architectural layers:

```
Presentation Layer:
  pages/, views/, templates/, components/, screens/, routes/

Business Logic Layer:
  services/, domain/, core/, business/, logic/, use-cases/

Data Access Layer:
  models/, repositories/, dal/, data/, database/, persistence/

Infrastructure Layer:
  infrastructure/, infra/, adapters/, external/, clients/

Shared/Common:
  utils/, helpers/, common/, shared/, lib/, types/
```

### Data Flow Tracing

```bash
# Trace imports from entry point (Python)
grep -rn "^from\|^import" app/main.py | head -20

# Trace imports from entry point (TypeScript)
grep -rn "^import" src/index.ts | head -20

# Find who imports a specific module
grep -rn "from.*models" --include="*.py" | sed 's/:.*//g' | sort -u

# Trace HTTP request flow
grep -rn "@app\.\(get\|post\|put\|delete\)\|@router\.\|HandleFunc\|app\.use" \
  --include="*.py" --include="*.ts" --include="*.go" | head -30
```

## Phase 3: Convention Mapping — Deep Dive

### Naming Pattern Extraction

```bash
# File naming conventions
find . -name '*.py' -not -path '*/node_modules/*' | xargs basename -a | sort -u | head -30
find . -name '*.ts' -not -path '*/node_modules/*' | xargs basename -a | sort -u | head -30

# Class naming patterns
grep -rn "^class " --include="*.py" | sed 's/.*class //' | sed 's/[(:].*//' | head -20
grep -rn "export class\|export default class" --include="*.ts" | sed 's/.*class //' | sed 's/[{ ].*//' | head -20

# Function naming patterns
grep -rn "^def \|^async def " --include="*.py" | sed 's/.*def //' | sed 's/(.*//' | head -20
grep -rn "export function\|export async function" --include="*.ts" | sed 's/.*function //' | sed 's/(.*//' | head -20
```

### Error Handling Pattern Detection

```bash
# Python error handling
grep -rn "try:\|except \|raise " --include="*.py" | head -20

# TypeScript/JavaScript error handling
grep -rn "try {\|catch (\|throw new" --include="*.ts" --include="*.js" | head -20

# Go error handling
grep -rn "if err != nil\|errors\.\|fmt\.Errorf" --include="*.go" | head -20

# Custom error classes
grep -rn "class.*Error\|class.*Exception" --include="*.py" --include="*.ts" | head -20
```

### Logging Pattern Detection

```bash
# Python logging
grep -rn "logger\.\|logging\.\|log\." --include="*.py" | head -20

# JavaScript/TypeScript logging
grep -rn "console\.\|logger\.\|winston\.\|pino\." --include="*.ts" --include="*.js" | head -20

# Structured logging detection
grep -rn "structlog\|bunyan\|pino\|slog\." --include="*.py" --include="*.ts" --include="*.go" | head -10
```

## Phase 4: Dependency Graph — Deep Dive

### External Dependency Inventory

```bash
# Node.js dependencies
jq '.dependencies, .devDependencies' package.json 2>/dev/null

# Python dependencies
cat requirements.txt 2>/dev/null || cat pyproject.toml 2>/dev/null | grep -A 50 '\[project.dependencies\]'

# Go dependencies
cat go.mod 2>/dev/null | grep -E "^\t"

# Rust dependencies
grep -A 1 '\[dependencies\]' Cargo.toml 2>/dev/null
```

### Internal Dependency Mapping

```bash
# Python: which modules import which
grep -rn "^from " --include="*.py" | awk -F'from ' '{print $2}' | awk '{print $1}' | sort | uniq -c | sort -rn | head -20

# TypeScript: most-imported internal modules
grep -rn "from '\.\|from \"\." --include="*.ts" | sed "s/.*from ['\"]//;s/['\"].*//" | sort | uniq -c | sort -rn | head -20

# Go: internal package imports
grep -rn "\".*/" --include="*.go" | grep -v "vendor\|node_modules" | sed 's/.*"//;s/".*//' | sort | uniq -c | sort -rn | head -20
```

## Phase 5: Quality Snapshot — Deep Dive

### Complexity Indicators

```bash
# Largest files by line count (potential god classes)
find . -type f \( -name '*.py' -o -name '*.ts' -o -name '*.js' -o -name '*.go' -o -name '*.rs' \) \
  -not -path '*/node_modules/*' -not -path '*/vendor/*' \
  -exec wc -l {} + | sort -rn | head -15

# Longest functions (Python — crude heuristic)
grep -rn "^def \|^    def \|^async def " --include="*.py" | wc -l

# Files with most functions (responsibility concentration)
for f in $(find . -name '*.py' -not -path '*/node_modules/*'); do
  count=$(grep -c "^def \|^    def " "$f" 2>/dev/null)
  [ "$count" -gt 10 ] && echo "$count $f"
done | sort -rn | head -10
```

### Git-Based Health Metrics

```bash
# Most frequently changed files (churn hotspots)
git log --name-only --pretty=format: --since="90 days ago" \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -15

# Contributors per file (bus factor)
git log --format='%aN' -- src/ | sort | uniq -c | sort -rn | head -10

# Files not touched in 1+ year (staleness)
git log --diff-filter=M --name-only --since="1 year ago" --pretty=format: \
  | sort -u > /tmp/recently_changed.txt
find . -name '*.py' -o -name '*.ts' -o -name '*.js' | sort > /tmp/all_files.txt
comm -23 /tmp/all_files.txt /tmp/recently_changed.txt | head -20

# Commit frequency (project activity)
git log --oneline --since="30 days ago" | wc -l
git log --oneline --since="90 days ago" | wc -l
git log --oneline --since="365 days ago" | wc -l
```

### Test Coverage Indicators

```bash
# Test file ratio
total=$(find . -name '*.py' -o -name '*.ts' -o -name '*.js' | grep -v node_modules | wc -l)
tests=$(find . -name '*test*' -o -name '*spec*' | grep -v node_modules | wc -l)
echo "Test files: $tests / Total: $total (ratio: $(echo "scale=2; $tests * 100 / $total" | bc)%)"

# Test framework detection
grep -rl "import pytest\|from pytest" --include="*.py" | head -3 && echo "pytest detected"
grep -rl "describe(\|it(\|test(" --include="*.ts" --include="*.js" | head -3 && echo "jest/vitest detected"
grep -rl "testing.T\|func Test" --include="*.go" | head -3 && echo "go testing detected"

# Untested modules (no corresponding test file)
for f in $(find src/ -name '*.ts' -not -name '*.test.*' -not -name '*.spec.*' 2>/dev/null); do
  base=$(basename "$f" .ts)
  find . -name "${base}.test.ts" -o -name "${base}.spec.ts" | grep -q . || echo "UNTESTED: $f"
done | head -20
```
