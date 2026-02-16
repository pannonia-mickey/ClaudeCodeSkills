---
name: Codebase Quality
description: This skill should be used when the user asks about "code quality", "technical debt", "complexity metrics", "code smells", "test coverage", "documentation coverage", "code freshness", "hotspot analysis", "git churn", or "quality assessment". It covers complexity measurement, coupling analysis, test and documentation coverage, technical debt identification, and remediation planning.
---

# Code Quality Assessment

## Complexity Metrics

### Lines of Code (LOC)

```bash
# Total source lines (excluding blanks and comments)
find . -type f \( -name '*.py' -o -name '*.ts' -o -name '*.js' -o -name '*.go' -o -name '*.rs' \) \
  -not -path '*/node_modules/*' -not -path '*/vendor/*' -not -path '*/venv/*' \
  -exec cat {} + | grep -v '^\s*$' | grep -v '^\s*//' | grep -v '^\s*#' | wc -l

# LOC per file (identify outliers)
find . -type f \( -name '*.py' -o -name '*.ts' -o -name '*.js' \) \
  -not -path '*/node_modules/*' -not -path '*/vendor/*' \
  -exec wc -l {} + | sort -rn | head -15

# LOC distribution summary
find . -type f -name '*.ts' -not -path '*/node_modules/*' -exec wc -l {} + \
  | awk 'END{print "Total: "NR-1" files"} $1>500{big++} $1>200{med++} $1<=200{small++} \
  END{print "Large (>500): "big"\nMedium (200-500): "med-big"\nSmall (<200): "small}'
```

### Function Count and Size

```bash
# Functions per file (Python)
for f in $(find . -name '*.py' -not -path '*/venv/*'); do
  count=$(grep -c "^\s*def \|^\s*async def " "$f" 2>/dev/null)
  [ "$count" -gt 0 ] && echo "$count functions: $f"
done | sort -rn | head -15

# Functions per file (TypeScript)
for f in $(find . -name '*.ts' -not -path '*/node_modules/*' -not -name '*.test.*'); do
  count=$(grep -cE "^\s*(export )?(async )?function |^\s*(public|private|protected|static)?\s*(async )?\w+\(" "$f" 2>/dev/null)
  [ "$count" -gt 0 ] && echo "$count functions: $f"
done | sort -rn | head -15
```

### Nesting Depth (Cognitive Complexity Proxy)

```bash
# Deep nesting indicator — lines with high indentation
# Python (4-space indent, depth > 4 = 16+ spaces)
grep -rn "^                    " --include="*.py" | wc -l

# TypeScript (2-space indent, depth > 5 = 10+ spaces)
grep -rn "^          " --include="*.ts" | grep -v "node_modules" | wc -l

# Files with deepest nesting
for f in $(find . -name '*.ts' -not -path '*/node_modules/*'); do
  depth=$(awk '{match($0, /^[[:space:]]*/); if (RLENGTH > max) max=RLENGTH} END{print max/2}' "$f" 2>/dev/null)
  [ "${depth%.*}" -gt 6 ] && echo "Depth $depth: $f"
done | sort -rn | head -10
```

## Coupling and Cohesion Analysis

### Afferent Coupling (Ca) — Who Depends on This Module

```bash
# Count how many files import each module (TypeScript)
for dir in $(find src/ -maxdepth 1 -type d 2>/dev/null); do
  module=$(basename "$dir")
  count=$(grep -rn "from.*/$module/" --include="*.ts" | grep -v "$dir" | wc -l)
  echo "$count dependents: $module"
done | sort -rn | head -10

# High Ca = many dependents = hard to change (stable but risky)
```

### Efferent Coupling (Ce) — What This Module Depends On

```bash
# Count external imports per module
for dir in $(find src/ -maxdepth 1 -type d 2>/dev/null); do
  module=$(basename "$dir")
  count=$(grep -rn "^import\|^from" "$dir" --include="*.ts" --include="*.py" 2>/dev/null \
    | grep -v "from '\.\|from \"\.\|from \.\." | wc -l)
  echo "$count dependencies: $module"
done | sort -rn | head -10

# High Ce = many dependencies = fragile, breaks when deps change
```

### Instability Index

```
Instability = Ce / (Ca + Ce)
- 0 = maximally stable (many dependents, few dependencies)
- 1 = maximally unstable (few dependents, many dependencies)
```

## Test Coverage Assessment

### Test Presence and Ratio

```bash
# Count source vs test files
source=$(find . -name '*.ts' -not -name '*.test.*' -not -name '*.spec.*' \
  -not -path '*/node_modules/*' | wc -l)
tests=$(find . -name '*.test.*' -o -name '*.spec.*' | grep -v node_modules | wc -l)
echo "Source: $source, Tests: $tests, Ratio: $(echo "scale=1; $tests * 100 / $source" | bc)%"

# Find modules with zero test coverage
for dir in $(find src/ -maxdepth 1 -type d 2>/dev/null); do
  src_count=$(find "$dir" -name '*.ts' -not -name '*.test.*' -not -name '*.spec.*' | wc -l)
  test_count=$(find "$dir" -name '*.test.*' -o -name '*.spec.*' 2>/dev/null | wc -l)
  [ "$src_count" -gt 0 ] && [ "$test_count" -eq 0 ] && echo "NO TESTS: $dir ($src_count source files)"
done
```

### Test Framework Detection

```bash
# Detect test framework
grep -l "jest\|vitest\|mocha\|jasmine" package.json 2>/dev/null
grep -l "pytest\|unittest\|nose" pyproject.toml setup.cfg 2>/dev/null
grep -l "testing" go.mod 2>/dev/null

# Check for coverage configuration
grep -rn "coverage\|istanbul\|c8\|v8" package.json vitest.config.* jest.config.* 2>/dev/null | head -5
grep -rn "coverage\|cov" pyproject.toml setup.cfg 2>/dev/null | head -5
```

## Documentation Coverage

```bash
# README presence
find . -maxdepth 1 -name 'README*' -exec echo "Root README: {}" \;
find . -maxdepth 2 -name 'README*' | wc -l

# API documentation
find . -maxdepth 3 -type d -name 'docs' -o -name 'documentation' -o -name 'api-docs'
find . -name 'openapi*' -o -name 'swagger*' -o -name '*.apib' | head -5

# Inline documentation density (Python docstrings)
total_funcs=$(grep -rn "^\s*def " --include="*.py" | wc -l)
documented=$(grep -rn "^\s*def " --include="*.py" -A 1 | grep '"""' | wc -l)
echo "Documented functions: $documented / $total_funcs"

# JSDoc coverage (TypeScript)
total_exports=$(grep -rn "^export " --include="*.ts" | wc -l)
documented=$(grep -rn "^\s*/\*\*" --include="*.ts" | wc -l)
echo "JSDoc blocks: $documented, Exports: $total_exports"

# Contributing guide
find . -maxdepth 1 -name 'CONTRIBUTING*' -o -name 'DEVELOPMENT*'
```

## Technical Debt Identification

### Code Smell Detection

```bash
# TODO/FIXME/HACK/WORKAROUND markers
grep -rn "TODO\|FIXME\|HACK\|WORKAROUND\|XXX\|TEMP" \
  --include="*.ts" --include="*.py" --include="*.go" --include="*.rs" \
  | grep -v node_modules | wc -l

# List TODO/FIXME with context
grep -rn "TODO\|FIXME" --include="*.ts" --include="*.py" \
  | grep -v node_modules | head -20

# Suppressed linter warnings (deferred debt)
grep -rn "eslint-disable\|noqa\|nolint\|#allow\|@SuppressWarnings" \
  --include="*.ts" --include="*.py" --include="*.go" --include="*.java" \
  | grep -v node_modules | wc -l
```

### Code Freshness (Git Churn)

```bash
# Most frequently changed files (last 90 days) — churn hotspots
git log --name-only --pretty=format: --since="90 days ago" \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -15

# Files changed AND are large — highest debt risk
git log --name-only --pretty=format: --since="90 days ago" \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -30 \
  | while read count file; do
    [ -f "$file" ] && lines=$(wc -l < "$file") && [ "$lines" -gt 200 ] \
      && echo "$count changes, $lines lines: $file"
  done | head -10

# Files not modified in 1+ year (potential stale code)
git log --diff-filter=M --name-only --pretty=format: --since="1 year ago" \
  | sort -u > /tmp/recent.txt
find . -name '*.ts' -o -name '*.py' | grep -v node_modules | sort > /tmp/all.txt
comm -23 /tmp/all.txt /tmp/recent.txt | head -20
```

## Quality Report Compilation

Compile all metrics into the Quality Assessment Report format:

| Category | Metric | Value | Rating |
|----------|--------|-------|--------|
| Size | Total LOC | — | — |
| Size | Avg file size | — | Good (<200) / Fair (200-500) / Poor (>500) |
| Size | Max file size | — | — |
| Complexity | Deep nesting files | — | Good (0) / Fair (1-5) / Poor (>5) |
| Testing | Test file ratio | — | Good (>40%) / Fair (20-40%) / Poor (<20%) |
| Testing | Untested modules | — | — |
| Docs | README present | — | Yes/No |
| Docs | API docs | — | Yes/No |
| Debt | TODO/FIXME count | — | Good (<10) / Fair (10-50) / Poor (>50) |
| Debt | Lint suppressions | — | Good (<5) / Fair (5-20) / Poor (>20) |
| Activity | Commits (30d) | — | Active (>20) / Moderate (5-20) / Low (<5) |
| Activity | Churn hotspots | — | List top 5 |

## References

- [Metrics and Indicators](references/metrics-and-indicators.md) — Detailed code quality metrics at file, module, and project levels including git-based analysis techniques.
- [Tech Debt Assessment](references/tech-debt-assessment.md) — Technical debt classification framework, severity assessment, prioritization matrix, and remediation planning patterns.
