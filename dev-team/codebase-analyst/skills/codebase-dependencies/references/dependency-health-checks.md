# Dependency Health Checks

## Vulnerability Scanning — Detailed Guide

### Node.js (npm/pnpm/yarn)

```bash
# Full audit with severity filtering
npm audit --audit-level=high
npm audit --json | jq '.vulnerabilities | to_entries[] | {name: .key, severity: .value.severity, via: .value.via[0].title}'

# Fix automatically (safe patches only)
npm audit fix

# Fix with breaking changes (review carefully)
npm audit fix --force

# pnpm audit
pnpm audit --audit-level=high
```

### Python (pip-audit / safety)

```bash
# pip-audit (recommended — uses OSV database)
pip-audit --strict --desc

# safety (uses Safety DB)
safety check --full-report

# Check specific requirements file
pip-audit -r requirements.txt

# Output as JSON for CI integration
pip-audit --format json --output audit-results.json
```

### Rust (cargo-audit)

```bash
# Install and run
cargo install cargo-audit
cargo audit

# With fix suggestions
cargo audit fix --dry-run

# Check for yanked crates too
cargo audit --deny yanked
```

### Go (govulncheck)

```bash
# Install and run
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...

# Verbose output with call stacks
govulncheck -v ./...

# Check specific module
govulncheck -test ./...
```

### .NET

```bash
# Built-in vulnerability check
dotnet list package --vulnerable --include-transitive

# With specific feed
dotnet list package --vulnerable --source https://api.nuget.org/v3/index.json
```

## Outdated Dependency Detection

### Categorizing Outdated Packages

```bash
# Node.js — detailed outdated report
npm outdated --long 2>/dev/null

# Output format:
# Package  Current  Wanted  Latest  Location  Package Type
# react    17.0.2   17.0.2  18.2.0  myapp     dependencies

# Categorize by semver gap
npm outdated --json 2>/dev/null | jq '
  to_entries[] | {
    name: .key,
    current: .value.current,
    latest: .value.latest,
    gap: (
      if (.value.current | split(".")[0]) != (.value.latest | split(".")[0]) then "MAJOR"
      elif (.value.current | split(".")[1]) != (.value.latest | split(".")[1]) then "MINOR"
      else "PATCH"
      end
    )
  }
'
```

### Python Outdated Check

```bash
# List outdated packages
pip list --outdated --format=json | jq '.[] | {name: .name, current: .version, latest: .latest_version}'

# Check against constraints
pip-compile --upgrade --dry-run requirements.in 2>/dev/null
```

## Abandoned Package Identification

### Heuristics for Abandoned Packages

A package may be abandoned if:
- No release in 2+ years
- Many open issues with no maintainer response
- Deprecated notice in README or npm
- Archived repository on GitHub

### Detection Commands

```bash
# Node.js — check last publish date
for pkg in $(jq -r '.dependencies | keys[]' package.json 2>/dev/null); do
  date=$(npm view "$pkg" time.modified 2>/dev/null | head -1)
  echo "$date $pkg"
done | sort | head -20

# Check for deprecated packages
for pkg in $(jq -r '.dependencies | keys[]' package.json 2>/dev/null); do
  deprecated=$(npm view "$pkg" deprecated 2>/dev/null)
  [ -n "$deprecated" ] && echo "DEPRECATED: $pkg — $deprecated"
done

# Python — check last upload date on PyPI
for pkg in $(cat requirements.txt 2>/dev/null | sed 's/[>=<].*//' | grep -v '^#'); do
  echo "Checking $pkg..."
  curl -s "https://pypi.org/pypi/$pkg/json" 2>/dev/null \
    | jq -r '.info.version as $v | .releases[$v][-1].upload_time_iso_8601' 2>/dev/null
done
```

## Transitive Dependency Analysis

### Why Transitive Dependencies Matter

Direct dependencies bring in their own dependencies (transitives). A project with 50 direct dependencies may have 500+ transitive ones — each is an attack surface.

### Analysis Commands

```bash
# Node.js — full dependency tree
npm ls --all --depth=5 2>/dev/null | head -50

# Count total vs direct
direct=$(jq '.dependencies | length' package.json 2>/dev/null)
total=$(ls node_modules/ 2>/dev/null | wc -l)
echo "Direct: $direct, Total (including transitive): $total, Ratio: $(echo "$total / $direct" | bc)x"

# Python — dependency tree
pipdeptree 2>/dev/null | head -50

# Rust — dependency tree
cargo tree 2>/dev/null | head -50

# Go — dependency graph
go mod graph 2>/dev/null | head -50
```

### Finding Problematic Transitives

```bash
# Packages that appear multiple times (version conflicts)
npm ls --all 2>/dev/null | grep -oP '[\w@/-]+@[\d.]+' | sort | uniq -c | sort -rn \
  | awk '$1 > 1 {print}' | head -15

# Duplicate packages with different versions
npm ls --all 2>/dev/null | grep "deduped\|invalid" | head -20
```

## Lock File Health Checks

### Why Lock Files Matter

Lock files ensure deterministic installs. Missing or outdated lock files cause "works on my machine" problems.

```bash
# Verify lock file exists
test -f package-lock.json && echo "PASS: package-lock.json exists" || echo "FAIL: missing lock file"
test -f pnpm-lock.yaml && echo "PASS: pnpm-lock.yaml exists" || echo "FAIL: missing lock file"
test -f poetry.lock && echo "PASS: poetry.lock exists" || echo "FAIL: missing lock file"
test -f Cargo.lock && echo "PASS: Cargo.lock exists" || echo "FAIL: missing lock file"
test -f go.sum && echo "PASS: go.sum exists" || echo "FAIL: missing lock file"

# Check lock file is committed (not in .gitignore)
grep -q "package-lock.json\|pnpm-lock.yaml\|yarn.lock" .gitignore 2>/dev/null \
  && echo "WARNING: lock file in .gitignore" || echo "PASS: lock file not ignored"

# Verify lock file is in sync with manifest
npm ci --dry-run 2>/dev/null || echo "Lock file may be out of sync"
```

## Security Advisory Databases

| Database | URL | Ecosystems |
|----------|-----|------------|
| GitHub Advisory Database | github.com/advisories | All |
| OSV (Open Source Vulnerabilities) | osv.dev | All |
| npm Advisory Database | npmjs.com/advisories | Node.js |
| PyPI Advisory Database | pypi.org/security | Python |
| RustSec Advisory Database | rustsec.org | Rust |
| Go Vulnerability Database | vuln.go.dev | Go |
| NVD (NIST) | nvd.nist.gov | All (CVE-based) |

### Querying Advisories

```bash
# Query OSV API for a specific package
curl -s "https://api.osv.dev/v1/query" \
  -d '{"package": {"name": "lodash", "ecosystem": "npm"}}' \
  | jq '.vulns | length'

# GitHub Advisory API (requires gh CLI)
gh api graphql -f query='
  { securityAdvisories(first: 5, ecosystem: NPM, orderBy: {field: PUBLISHED_AT, direction: DESC}) {
    nodes { summary severity publishedAt }
  } }
'
```
