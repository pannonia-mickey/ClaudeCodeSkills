# Metrics and Indicators

## File-Level Metrics

### Lines of Code (LOC)

| Rating | LOC | Interpretation |
|--------|-----|---------------|
| Good | < 200 | Focused, single responsibility |
| Fair | 200–500 | Acceptable but review scope |
| Poor | 500–1000 | Likely multiple responsibilities |
| Critical | > 1000 | God class, refactor immediately |

```bash
# File size distribution
find . -type f -name '*.ts' -not -path '*/node_modules/*' -exec wc -l {} + \
  | awk '{
    if ($1 <= 200) small++
    else if ($1 <= 500) medium++
    else if ($1 <= 1000) large++
    else huge++
  } END {
    print "Small (<200): " small
    print "Medium (200-500): " medium
    print "Large (500-1000): " large
    print "Huge (>1000): " huge
  }'
```

### Function Count Per File

| Rating | Functions | Interpretation |
|--------|-----------|---------------|
| Good | 1–10 | Focused module |
| Fair | 11–20 | Getting complex |
| Poor | > 20 | Too many responsibilities |

```bash
# Python
for f in $(find . -name '*.py' -not -path '*/venv/*'); do
  count=$(grep -c "^\s*def " "$f" 2>/dev/null)
  [ "$count" -gt 10 ] && echo "$count: $f"
done | sort -rn | head -10

# TypeScript
for f in $(find . -name '*.ts' -not -path '*/node_modules/*'); do
  count=$(grep -cE "^\s*(export )?(async )?function |^\s*(public|private).*\(" "$f" 2>/dev/null)
  [ "$count" -gt 10 ] && echo "$count: $f"
done | sort -rn | head -10
```

### Import Count (Coupling Indicator)

| Rating | Imports | Interpretation |
|--------|---------|---------------|
| Good | 1–8 | Normal coupling |
| Fair | 9–15 | High coupling |
| Poor | > 15 | Excessive coupling, consider splitting |

```bash
for f in $(find . -name '*.ts' -not -path '*/node_modules/*'); do
  count=$(grep -c "^import " "$f" 2>/dev/null)
  [ "$count" -gt 10 ] && echo "$count imports: $f"
done | sort -rn | head -10
```

## Module-Level Metrics

### Afferent Coupling (Ca) — Incoming Dependencies

How many other modules depend on this module. High Ca means the module is widely used and changes are risky.

```bash
for dir in $(find src/ -maxdepth 1 -type d 2>/dev/null); do
  module=$(basename "$dir")
  ca=$(grep -rn "from.*/$module" --include="*.ts" | grep -v "$dir" | wc -l)
  echo "Ca=$ca: $module"
done | sort -t= -k2 -rn | head -10
```

### Efferent Coupling (Ce) — Outgoing Dependencies

How many other modules this module depends on. High Ce means the module is fragile and breaks when dependencies change.

```bash
for dir in $(find src/ -maxdepth 1 -type d 2>/dev/null); do
  module=$(basename "$dir")
  ce=$(grep -rn "from '\.\./\|from \"\.\./" "$dir" --include="*.ts" 2>/dev/null \
    | sed "s/.*from ['\"]\.\.\/\([^/]*\).*/\1/" | sort -u | wc -l)
  echo "Ce=$ce: $module"
done | sort -t= -k2 -rn | head -10
```

### Instability (I) = Ce / (Ca + Ce)

| I Value | Interpretation | Guidance |
|---------|---------------|----------|
| 0.0 | Maximally stable | Many dependents, few dependencies. Hard to change. |
| 0.0–0.3 | Stable | Core modules, shared utilities |
| 0.3–0.7 | Balanced | Application services |
| 0.7–1.0 | Unstable | Leaf modules, implementations |
| 1.0 | Maximally unstable | No dependents. Easy to change. |

### Abstractness (A) — Interface vs Implementation Ratio

```bash
# TypeScript: interfaces vs classes per module
for dir in $(find src/ -maxdepth 1 -type d 2>/dev/null); do
  module=$(basename "$dir")
  interfaces=$(grep -rn "^export interface\|^export type\|^export abstract" "$dir" --include="*.ts" 2>/dev/null | wc -l)
  classes=$(grep -rn "^export class" "$dir" --include="*.ts" 2>/dev/null | wc -l)
  total=$((interfaces + classes))
  [ "$total" -gt 0 ] && echo "A=$(echo "scale=2; $interfaces / $total" | bc): $module ($interfaces interfaces, $classes classes)"
done | sort -t= -k2 -rn | head -10
```

### Distance from Main Sequence

```
D = |A + I - 1|

Ideal: D = 0 (on the main sequence line)
High D: Module is in the "zone of pain" (stable but concrete)
        or "zone of uselessness" (unstable but abstract)
```

## Project-Level Metrics

### Test Ratio

| Rating | Ratio | Interpretation |
|--------|-------|---------------|
| Good | > 40% | Well-tested codebase |
| Fair | 20–40% | Moderate coverage |
| Poor | < 20% | Significant testing gaps |

```bash
source_files=$(find . \( -name '*.ts' -o -name '*.py' \) -not -name '*.test.*' -not -name '*.spec.*' \
  -not -path '*/node_modules/*' -not -path '*/venv/*' | wc -l)
test_files=$(find . \( -name '*.test.*' -o -name '*.spec.*' -o -name 'test_*' \) \
  -not -path '*/node_modules/*' | wc -l)
echo "Test ratio: $(echo "scale=1; $test_files * 100 / $source_files" | bc)% ($test_files test files / $source_files source files)"
```

### Documentation Coverage

```bash
# README count
readmes=$(find . -maxdepth 3 -name 'README*' | wc -l)
echo "README files: $readmes"

# API documentation
api_docs=$(find . -name 'openapi*' -o -name 'swagger*' -o -name '*.apib' 2>/dev/null | wc -l)
echo "API spec files: $api_docs"

# Inline documentation (Python docstrings)
funcs=$(grep -rn "^\s*def " --include="*.py" | wc -l)
docstrings=$(grep -rn "^\s*def " --include="*.py" -A 1 | grep '"""' | wc -l)
echo "Python docstring coverage: $(echo "scale=1; $docstrings * 100 / $funcs" | bc)%"
```

### Dependency Metrics

| Metric | Good | Fair | Poor |
|--------|------|------|------|
| Direct deps | < 30 | 30–60 | > 60 |
| Transitive ratio | < 5x | 5–10x | > 10x |
| Outdated % | < 10% | 10–30% | > 30% |
| Vulnerable | 0 | 1–3 low | Any high/critical |

## Git-Based Metrics

### Churn Analysis (Change Frequency)

```bash
# Most changed files in last 90 days
git log --name-only --pretty=format: --since="90 days ago" \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -15

# Churn + complexity = highest risk files
git log --name-only --pretty=format: --since="90 days ago" \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -50 \
  | while read changes file; do
    [ -f "$file" ] && loc=$(wc -l < "$file") \
      && risk=$(echo "$changes * $loc" | bc) \
      && echo "risk=$risk changes=$changes loc=$loc $file"
  done | sort -t= -k2 -rn | head -10
```

### Hotspot Detection

Files that are both large AND frequently changed are the highest-risk hotspots.

```
Risk Score = Change Frequency x File Size (LOC)

High Risk:  > 50 changes AND > 500 LOC
Medium Risk: > 20 changes AND > 200 LOC
Low Risk:   Everything else
```

### Contributor Distribution (Bus Factor)

```bash
# Contributors per file (bus factor analysis)
for f in $(find src/ -name '*.ts' -not -path '*/node_modules/*' | head -50); do
  contributors=$(git log --format='%aN' -- "$f" 2>/dev/null | sort -u | wc -l)
  [ "$contributors" -le 1 ] && echo "BUS FACTOR 1: $f"
done | head -15

# Overall contributor distribution
git shortlog -sn --since="1 year ago" | head -10

# Knowledge concentration
total_commits=$(git log --oneline --since="1 year ago" | wc -l)
top_contributor=$(git shortlog -sn --since="1 year ago" | head -1 | awk '{print $1}')
echo "Top contributor: $top_contributor / $total_commits total ($(echo "scale=1; $top_contributor * 100 / $total_commits" | bc)%)"
```

### Commit Frequency (Project Activity)

| Window | Active | Moderate | Low | Stale |
|--------|--------|----------|-----|-------|
| 30 days | > 30 | 10–30 | 1–9 | 0 |
| 90 days | > 100 | 30–100 | 1–29 | 0 |

```bash
echo "Last 7 days:   $(git log --oneline --since='7 days ago' | wc -l) commits"
echo "Last 30 days:  $(git log --oneline --since='30 days ago' | wc -l) commits"
echo "Last 90 days:  $(git log --oneline --since='90 days ago' | wc -l) commits"
echo "Last 365 days: $(git log --oneline --since='365 days ago' | wc -l) commits"
```

### Code Age Analysis

```bash
# Average age of source files (days since last modification)
for f in $(find src/ -name '*.ts' -not -path '*/node_modules/*' | head -50); do
  last_modified=$(git log -1 --format='%ct' -- "$f" 2>/dev/null)
  [ -n "$last_modified" ] && age=$(( ($(date +%s) - last_modified) / 86400 )) \
    && echo "$age days: $f"
done | sort -rn | head -15
```
