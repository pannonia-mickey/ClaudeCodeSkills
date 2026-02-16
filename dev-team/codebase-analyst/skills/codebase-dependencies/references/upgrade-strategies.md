# Upgrade Strategies

## Semver Impact Analysis

### Understanding Version Gaps

```
MAJOR.MINOR.PATCH — 2.1.3
│       │      └── Bug fixes, security patches (safe to upgrade)
│       └────────── New features, no breaking changes (usually safe)
└────────────────── Breaking changes (requires code migration)
```

### Assessing Upgrade Risk

| Gap Type | Risk | Example | Approach |
|----------|------|---------|----------|
| Patch | Low | 1.2.3 → 1.2.5 | Auto-merge, run tests |
| Minor | Low-Medium | 1.2.3 → 1.4.0 | Review changelog, run tests |
| Major | High | 1.2.3 → 2.0.0 | Read migration guide, plan changes |
| Multiple majors | Very High | 1.x → 3.x | Staged upgrade through each major |

```bash
# Categorize all outdated packages by risk level
npm outdated --json 2>/dev/null | jq '
  to_entries[] | {
    package: .key,
    current: .value.current,
    latest: .value.latest,
    risk: (
      if (.value.current | split(".")[0]) != (.value.latest | split(".")[0]) then "HIGH-MAJOR"
      elif (.value.current | split(".")[1]) != (.value.latest | split(".")[1]) then "MEDIUM-MINOR"
      else "LOW-PATCH"
      end
    )
  }
' | jq -s 'group_by(.risk) | map({risk: .[0].risk, count: length, packages: [.[].package]})'
```

## Staged Upgrade Approach

### Strategy: Patch → Minor → Major

Upgrade in order of increasing risk. Run the full test suite after each stage.

```bash
# Stage 1: Apply all patch upgrades
npm update 2>/dev/null   # Updates within semver range
npm test                 # Verify nothing breaks

# Stage 2: Apply minor upgrades one at a time
npm install package-a@^1.5.0   # Upgrade to latest minor
npm test

# Stage 3: Apply major upgrades one at a time
npm install package-b@^2.0.0   # Upgrade to next major
npm test
# Fix breaking changes before proceeding to next major upgrade
```

### Batch vs Individual Upgrades

| Approach | When to Use |
|----------|------------|
| Batch patches | Safe — apply all patches at once |
| Batch minors | Usually safe — apply all minors together |
| Individual majors | Always — one major upgrade at a time |
| Dependency groups | Related packages together (e.g., `@babel/*`) |

## Breaking Change Detection

### Pre-Upgrade Analysis

```bash
# Read changelog for breaking changes
# Node.js packages
npm view package-name changelog 2>/dev/null
# Or check GitHub releases page

# Check for migration guides
npm view package-name repository.url 2>/dev/null
# Visit: {repo-url}/blob/main/MIGRATION.md or CHANGELOG.md or UPGRADING.md

# TypeScript: check for type changes
# After upgrading, run type checker
npx tsc --noEmit 2>&1 | head -30

# Python: check deprecation warnings
python -W all -m pytest 2>&1 | grep -i "deprecat" | head -10
```

### Common Breaking Change Categories

| Category | Detection | Example |
|----------|-----------|---------|
| Removed API | `TypeScript errors`, `ImportError`, `undefined is not a function` | `lodash.pluck` removed in v4 |
| Changed signature | Type errors, wrong argument count | `express.Router()` options changed |
| Default behavior change | Tests fail with different output | `jest` changed default config |
| Minimum runtime version | Build/startup failures | Package requires Node.js 18+ |
| Peer dependency change | Install warnings/errors | React 18 required as peer |

```bash
# Check peer dependency requirements
npm ls --all 2>/dev/null | grep "UNMET PEER" | head -10

# Check minimum engine requirements
jq '.engines' package.json 2>/dev/null
node --version
```

## Migration Guide Discovery

### Where to Find Migration Guides

```bash
# Check package repository for migration docs
pkg_repo=$(npm view package-name repository.url 2>/dev/null | sed 's/.*github.com\///')

# Common locations:
# - CHANGELOG.md or CHANGES.md
# - MIGRATION.md or UPGRADING.md
# - docs/migration/ directory
# - GitHub releases page (release notes)
# - Official documentation site

# Search GitHub for migration guide
gh search repos "$pkg_repo migration guide" --limit 5 2>/dev/null
```

### Common Migration Patterns

**Renamed exports:**
```typescript
// Before (v1)
import { oldName } from 'package';
// After (v2)
import { newName } from 'package';
```

**Changed configuration:**
```typescript
// Before (v1)
const config = { option: true };
// After (v2)
const config = { options: { feature: true } };
```

**Async conversion:**
```typescript
// Before (v1) — synchronous
const result = pkg.doSync();
// After (v2) — async
const result = await pkg.doAsync();
```

## Automated Upgrade Tools

### Renovate (Recommended)

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true
    },
    {
      "matchUpdateTypes": ["minor"],
      "automerge": true,
      "stabilityDays": 3
    },
    {
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "labels": ["breaking-change"]
    }
  ],
  "schedule": ["before 7am on Monday"]
}
```

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      production:
        patterns: ["*"]
        exclude-patterns: ["@types/*", "eslint*", "prettier*"]
        update-types: ["minor", "patch"]
      dev:
        patterns: ["@types/*", "eslint*", "prettier*"]
        update-types: ["minor", "patch"]
```

### npm-check-updates (ncu)

```bash
# Check for updates without modifying
npx npm-check-updates

# Update package.json (patches only)
npx npm-check-updates --target patch -u

# Update package.json (minors only)
npx npm-check-updates --target minor -u

# Interactive mode — choose which to upgrade
npx npm-check-updates --interactive

# Filter specific packages
npx npm-check-updates --filter "react*,next*"
```

## Rollback Strategies

### Immediate Rollback

```bash
# Node.js — restore from lock file
git checkout package.json package-lock.json
npm ci

# Python — restore from lock
git checkout requirements.txt
pip install -r requirements.txt

# Rust — restore from lock
git checkout Cargo.toml Cargo.lock
cargo build
```

### Version Pinning After Rollback

```bash
# Pin to known-good version
npm install package-name@1.2.3 --save-exact

# Python — pin exact version
echo "package-name==1.2.3" >> requirements.txt

# Add to .npmrc to always save exact
echo "save-exact=true" >> .npmrc
```

### Pre-Upgrade Safety Checklist

1. Ensure all tests pass before starting upgrades
2. Create a branch for the upgrade work
3. Commit the lock file before any changes
4. Upgrade one major version at a time
5. Run full test suite after each upgrade
6. Check for deprecation warnings
7. Review type checking output
8. Test in staging environment before merging
