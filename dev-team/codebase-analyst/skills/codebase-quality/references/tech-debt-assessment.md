# Technical Debt Assessment

## Debt Classification Framework

### Architecture Debt

Structural problems that affect the overall system design.

| Indicator | Detection | Severity |
|-----------|-----------|----------|
| Missing layer separation | Controllers import repositories directly | High |
| Circular dependencies | Module A imports B, B imports A | High |
| God service | Service file > 1000 LOC | High |
| Shared mutable state | Global variables, singletons with state | Medium |
| Missing abstractions | No interfaces/protocols between layers | Medium |

```bash
# Detect architecture debt indicators
# Layer violation: controllers importing repositories
grep -rn "import.*repository\|from.*repository" --include="*.ts" --include="*.py" \
  | grep -iE "controller|route|handler" | head -10

# God services
find . -name '*service*' -o -name '*Service*' | while read f; do
  [ -f "$f" ] && loc=$(wc -l < "$f") && [ "$loc" -gt 500 ] && echo "$loc LOC: $f"
done | sort -rn

# Global state
grep -rn "^let \|^var \|global " --include="*.ts" --include="*.py" \
  | grep -v "const\|node_modules\|test\|spec" | head -10
```

### Design Debt

Problems in class/module design that make code hard to extend or maintain.

| Indicator | Detection | Severity |
|-----------|-----------|----------|
| God class | Class with > 20 methods | High |
| Feature envy | Method uses another class's data more than its own | Medium |
| Primitive obsession | IDs, money, emails as plain strings/numbers | Medium |
| Long parameter lists | Functions with > 5 parameters | Medium |
| Inappropriate intimacy | Class accesses another's private details | Low |

```bash
# God classes
for f in $(find . -name '*.ts' -not -path '*/node_modules/*'); do
  methods=$(grep -cE "^\s+(async )?(public |private )?[a-z]\w*\(" "$f" 2>/dev/null)
  [ "$methods" -gt 20 ] && echo "$methods methods: $f"
done | sort -rn | head -10

# Long parameter lists
grep -rn "function.*,.*,.*,.*,.*," --include="*.ts" | grep -v node_modules | head -10
grep -rn "def .*,.*,.*,.*,.*," --include="*.py" | grep -v venv | head -10
```

### Code Debt

Implementation-level issues that reduce readability and maintainability.

| Indicator | Detection | Severity |
|-----------|-----------|----------|
| Duplicated logic | Same patterns repeated in multiple files | Medium |
| Magic numbers/strings | Hardcoded values without named constants | Low |
| Complex conditionals | Nested if/else > 3 levels deep | Medium |
| Dead code | Unused exports, commented-out code | Low |
| Inconsistent naming | Mixed conventions in same module | Low |

```bash
# TODO/FIXME/HACK markers (acknowledged code debt)
grep -rn "TODO\|FIXME\|HACK\|WORKAROUND\|XXX" \
  --include="*.ts" --include="*.py" --include="*.go" \
  | grep -v node_modules | wc -l

# Commented-out code blocks
grep -rn "^\s*//.*=\|^\s*//.*function\|^\s*#.*def " \
  --include="*.ts" --include="*.py" | grep -v node_modules | head -10

# Magic numbers (numbers other than 0, 1, -1 in logic)
grep -rn "[^0-9][2-9][0-9]*[^0-9]" --include="*.ts" \
  | grep -vE "port|timeout|retry|max|min|limit|size|length|index|test|spec|config" \
  | head -10
```

### Test Debt

Missing or inadequate test coverage.

| Indicator | Detection | Severity |
|-----------|-----------|----------|
| No test framework | No test files, no test config | Critical |
| Zero coverage modules | Source directory with no tests | High |
| Missing edge case tests | Only happy path tested | Medium |
| Flaky tests | Tests that pass/fail intermittently | High |
| Slow test suite | Full suite > 5 minutes | Medium |

```bash
# Modules with no tests
for dir in $(find src/ -maxdepth 1 -type d 2>/dev/null); do
  module=$(basename "$dir")
  src=$(find "$dir" -name '*.ts' -not -name '*.test.*' -not -name '*.spec.*' | wc -l)
  tests=$(find "$dir" -name '*.test.*' -o -name '*.spec.*' 2>/dev/null | wc -l)
  [ "$src" -gt 0 ] && [ "$tests" -eq 0 ] && echo "NO TESTS: $module ($src source files)"
done

# Test suite execution time (if configured)
grep -rn "testTimeout\|timeout" jest.config.* vitest.config.* 2>/dev/null | head -5
```

### Documentation Debt

Missing or outdated documentation.

| Indicator | Detection | Severity |
|-----------|-----------|----------|
| No README | Missing root README | High |
| No API docs | No OpenAPI/Swagger spec | Medium |
| Outdated README | README doesn't match current code | Medium |
| No inline docs | Functions without docstrings/JSDoc | Low |
| No CONTRIBUTING guide | No contributor onboarding | Low |

```bash
# Documentation checklist
test -f README.md && echo "PASS: README.md" || echo "FAIL: No README"
test -f CONTRIBUTING.md && echo "PASS: CONTRIBUTING.md" || echo "MISSING: CONTRIBUTING.md"
find . -name 'openapi*' -o -name 'swagger*' | head -1 | grep . \
  && echo "PASS: API spec found" || echo "MISSING: No API spec"
```

### Infrastructure Debt

Build, deployment, and operational issues.

| Indicator | Detection | Severity |
|-----------|-----------|----------|
| No CI/CD | No pipeline configuration | High |
| No linting | No linter configuration | Medium |
| No type checking | No tsconfig strict mode | Medium |
| Outdated runtime | Node.js < LTS, Python < 3.10 | Medium |
| No containerization | No Dockerfile for deployable services | Low |

## Severity Assessment Framework

### Severity Levels

| Level | Impact | Response |
|-------|--------|----------|
| Critical | Blocks development or causes production issues | Fix this sprint |
| High | Significantly slows development velocity | Fix this quarter |
| Medium | Causes friction but workarounds exist | Plan for remediation |
| Low | Minor annoyance, cosmetic | Address opportunistically |

### Severity Scoring

```
Severity = Impact x Frequency x Blast Radius

Impact (1-5):     How much does this slow down work?
Frequency (1-5):  How often do developers encounter this?
Blast Radius (1-5): How many files/modules are affected?
```

| Score | Severity | Action |
|-------|----------|--------|
| 60–125 | Critical | Immediate attention |
| 30–59 | High | This quarter |
| 10–29 | Medium | Backlog with priority |
| 1–9 | Low | Opportunistic |

## Prioritization Matrix

### Impact vs Effort Matrix

```
                    Low Effort          High Effort
                ┌──────────────────┬──────────────────┐
  High Impact   │   QUICK WINS     │   MAJOR PROJECTS │
                │   Do first       │   Plan carefully  │
                │                  │                   │
                ├──────────────────┼──────────────────┤
  Low Impact    │   FILL-INS       │   THANKLESS TASKS│
                │   Do when free   │   Avoid or defer  │
                │                  │                   │
                └──────────────────┴──────────────────┘
```

### Quick Win Examples
- Add missing type annotations (low effort, prevents bugs)
- Delete dead code (low effort, reduces cognitive load)
- Add missing index to slow query (low effort, high performance gain)

### Major Project Examples
- Extract god class into domain services (high effort, high impact)
- Add test suite to untested module (medium effort, high impact)
- Migrate from legacy framework version (high effort, high impact)

## Remediation Planning

### Debt Inventory Template

```markdown
| ID | Category | Description | Severity | Location | Effort | Priority |
|----|----------|-------------|----------|----------|--------|----------|
| TD-001 | Architecture | Circular dep between auth and users | High | src/auth/, src/users/ | Medium | P1 |
| TD-002 | Test | No tests for payment module | Critical | src/payments/ | High | P0 |
| TD-003 | Code | Duplicated validation in 4 controllers | Medium | src/controllers/ | Low | P2 |
| TD-004 | Design | God class OrderService (45 methods) | High | src/services/order.ts | High | P1 |
```

### Sprint Integration

```markdown
## Debt Budget per Sprint
- Allocate 15-20% of sprint capacity to debt reduction
- Target 1-2 "Quick Wins" per sprint
- Target 1 "Major Project" per quarter
- Track debt score over time (should trend downward)

## Debt Tracking
- Tag debt items in issue tracker: `tech-debt`, `refactor`
- Review debt inventory monthly
- Report debt trends to stakeholders quarterly
- Celebrate debt reduction milestones
```

### Communication Template

```markdown
## Technical Debt Report — {Quarter}

### Summary
- Total debt items: {count}
- Critical: {n} | High: {n} | Medium: {n} | Low: {n}
- Items resolved this quarter: {n}
- New items identified: {n}
- Net change: {+/-n}

### Top 3 Risks
1. {Risk with business impact}
2. {Risk with business impact}
3. {Risk with business impact}

### Recommended Investments
1. {Action} — {estimated effort} — {expected benefit}
2. {Action} — {estimated effort} — {expected benefit}
```
