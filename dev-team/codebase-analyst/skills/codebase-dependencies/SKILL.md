---
name: Codebase Dependencies
description: This skill should be used when the user asks about "dependency analysis", "dependency audit", "outdated packages", "vulnerable dependencies", "dependency health", "package manifest", "license compliance", "bundle size", "dependency graph", or "upgrade strategy". It covers package manifest analysis, dependency health assessment, vulnerability scanning, and upgrade planning.
---

# Dependency Analysis

## Package Manifest Analysis

Read and interpret package manifests across ecosystems to inventory all dependencies.

### Ecosystem Detection

```bash
# Detect package ecosystem
find . -maxdepth 2 \
  -name 'package.json' \
  -o -name 'pyproject.toml' -o -name 'requirements.txt' -o -name 'Pipfile' \
  -o -name 'Cargo.toml' \
  -o -name 'go.mod' \
  -o -name '*.csproj' -o -name 'Directory.Packages.props' \
  -o -name 'pom.xml' -o -name 'build.gradle'
```

### Dependency Inventory

```bash
# Node.js — production vs dev dependencies
jq -r '.dependencies | keys[]' package.json 2>/dev/null | wc -l
jq -r '.devDependencies | keys[]' package.json 2>/dev/null | wc -l

# Python — from pyproject.toml
grep -A 100 '^\[project.dependencies\]' pyproject.toml 2>/dev/null | grep -B1 '^\[' | head -1
cat requirements.txt 2>/dev/null | grep -v '^#' | grep -v '^$' | wc -l

# Rust — from Cargo.toml
grep -c '=' Cargo.toml 2>/dev/null | head -1

# Go — from go.mod
grep -c "^\t" go.mod 2>/dev/null

# .NET — from .csproj
grep -c "PackageReference" *.csproj 2>/dev/null
```

### Version Constraint Analysis

| Constraint | Meaning | Risk |
|-----------|---------|------|
| `^1.2.3` | >=1.2.3, <2.0.0 | Low (npm default) |
| `~1.2.3` | >=1.2.3, <1.3.0 | Low |
| `>=1.2.3` | Any version >=1.2.3 | High (unbounded) |
| `*` or `latest` | Any version | Critical |
| `1.2.3` (exact) | Only 1.2.3 | Low (but misses patches) |

```bash
# Find loose version constraints (npm)
jq '.dependencies + .devDependencies | to_entries[] | select(.value | test("^[*>]|latest"))' \
  package.json 2>/dev/null

# Find unpinned Python dependencies
grep -v "==" requirements.txt 2>/dev/null | grep -v "^#" | grep -v "^$"
```

## Internal vs External Dependency Mapping

### Internal Module Dependencies

```bash
# TypeScript: internal imports between modules
grep -rn "from '@/\|from '\.\./" --include="*.ts" \
  | sed "s/.*from ['\"]//;s/['\"].*//" | sort | uniq -c | sort -rn | head -20

# Python: internal imports
grep -rn "^from \.\|^from app\.\|^from src\." --include="*.py" \
  | sed 's/.*from //;s/ import.*//' | sort | uniq -c | sort -rn | head -20

# Most-depended-on internal modules (bus factor for modules)
grep -rn "from " --include="*.ts" --include="*.py" \
  | sed "s/.*from ['\"]//;s/['\"].*//" | grep -v "node_modules" \
  | sort | uniq -c | sort -rn | head -15
```

### External Dependency Usage Analysis

```bash
# Most-imported external packages (TypeScript)
grep -rn "^import.*from '" --include="*.ts" \
  | sed "s/.*from '//;s/'.*//" | grep -v '^\.' | sort | uniq -c | sort -rn | head -15

# Most-imported external packages (Python)
grep -rn "^import \|^from " --include="*.py" \
  | sed 's/import //;s/from //;s/ .*//' | grep -v '^\.' \
  | sort | uniq -c | sort -rn | head -15
```

## Dependency Version Health

### Outdated Package Detection

```bash
# Node.js
npm outdated 2>/dev/null || pnpm outdated 2>/dev/null || yarn outdated 2>/dev/null

# Python
pip list --outdated 2>/dev/null

# Rust
cargo outdated 2>/dev/null

# Go
go list -m -u all 2>/dev/null | grep '\[' | head -20
```

### Vulnerability Scanning

```bash
# Node.js
npm audit 2>/dev/null || pnpm audit 2>/dev/null

# Python
pip-audit 2>/dev/null || safety check 2>/dev/null

# Rust
cargo audit 2>/dev/null

# Go
govulncheck ./... 2>/dev/null

# .NET
dotnet list package --vulnerable 2>/dev/null
```

## Dependency Graph Complexity

### Metrics

```bash
# Transitive dependency count (Node.js)
ls node_modules/ 2>/dev/null | wc -l

# Dependency tree depth
npm ls --all --depth=10 2>/dev/null | tail -5

# Duplicate packages
npm ls --all 2>/dev/null | grep "deduped" | wc -l

# Lock file size (indicator of transitive complexity)
wc -l package-lock.json pnpm-lock.yaml yarn.lock 2>/dev/null
```

### Dependency Weight Analysis (Frontend)

```bash
# Bundle analysis (if configured)
grep -rn "analyze\|bundle-analyzer\|webpack-bundle" package.json 2>/dev/null

# Check for tree-shaking indicators
grep -rn "\"sideEffects\"" package.json 2>/dev/null
grep -rn "\"module\"\|\"exports\"" package.json 2>/dev/null

# Large dependency detection
du -sh node_modules/*/ 2>/dev/null | sort -rh | head -15
```

## License Compliance

```bash
# Node.js license check
npx license-checker --summary 2>/dev/null

# Quick license scan from package.json files
find node_modules/ -maxdepth 2 -name 'package.json' \
  -exec grep -l '"license"' {} \; \
  | xargs grep '"license"' | sed 's/.*"license": "//;s/".*//' \
  | sort | uniq -c | sort -rn | head -15

# Python license check
pip-licenses 2>/dev/null | head -20

# Flag restrictive licenses
# GPL-3.0, AGPL-3.0, SSPL — may require source disclosure
# LGPL — acceptable if dynamically linked
# MIT, Apache-2.0, BSD — permissive, generally safe
```

## Dependency Assessment Report

After analysis, compile findings into a structured format:

```
## Dependency Summary
- Production deps: {count}
- Development deps: {count}
- Transitive deps: {count}
- Outdated: {count} ({percentage}%)
- Vulnerable: {count} (critical: {n}, high: {n}, medium: {n})

## Top Concerns
1. {Most critical vulnerability with CVE}
2. {Most outdated major version gap}
3. {Abandoned package with no updates in 2+ years}

## Recommendations
- Immediate: {patch vulnerabilities}
- Short-term: {upgrade outdated majors}
- Long-term: {replace abandoned packages}
```

## References

- [Dependency Health Checks](references/dependency-health-checks.md) — Detailed audit techniques including vulnerability database lookups, abandoned package identification, and transitive dependency analysis.
- [Upgrade Strategies](references/upgrade-strategies.md) — Staged upgrade approaches, breaking change detection, automated upgrade tools (Renovate, Dependabot), and rollback strategies.
