# Coverage Strategies

## Risk-Based Coverage Targets

```
┌─────────────────────────────────────────────────────┐
│ Risk Level │ Coverage │ Test Types                   │
├─────────────────────────────────────────────────────┤
│ Critical   │ 95%+     │ Unit + Integration + E2E     │
│ (payments, auth, data integrity)                     │
├─────────────────────────────────────────────────────┤
│ High       │ 85%+     │ Unit + Integration           │
│ (business logic, API endpoints, validations)         │
├─────────────────────────────────────────────────────┤
│ Medium     │ 75%+     │ Unit + Key Integration       │
│ (services, utilities, transformations)               │
├─────────────────────────────────────────────────────┤
│ Low        │ 60%+     │ Unit (happy path)            │
│ (display components, configuration, types)           │
└─────────────────────────────────────────────────────┘
```

## Mutation Testing

```typescript
// stryker.conf.mjs — mutation testing configuration
/** @type {import('@stryker-mutator/api/core').PartialStrykerOptions} */
export default {
  mutator: {
    plugins: ['@stryker-mutator/typescript-checker'],
    excludedMutations: ['StringLiteral', 'BlockStatement'],
  },
  testRunner: 'vitest',
  reporters: ['html', 'clear-text', 'progress'],
  coverageAnalysis: 'perTest',
  thresholds: {
    high: 80,
    low: 60,
    break: 50, // CI fails below this
  },
  // Focus on critical paths
  mutate: [
    'src/services/**/*.ts',
    '!src/services/**/*.test.ts',
    '!src/services/**/*.d.ts',
  ],
};

// Run: npx stryker run
// Mutation score = killed mutations / total mutations
// High score = tests catch real bugs, not just exercising code
```

## Incremental Coverage Enforcement

```yaml
# GitHub Actions — enforce coverage doesn't decrease
- name: Check coverage diff
  run: |
    # Get base branch coverage
    git fetch origin main
    git checkout origin/main -- coverage/coverage-summary.json 2>/dev/null || echo '{}' > coverage/base-coverage.json
    mv coverage/coverage-summary.json coverage/base-coverage.json 2>/dev/null || true

    # Run tests with coverage
    git checkout -
    npx vitest --coverage --reporter=json

    # Compare coverage
    node -e "
      const base = require('./coverage/base-coverage.json')?.total?.lines?.pct || 0;
      const current = require('./coverage/coverage-summary.json').total.lines.pct;
      console.log('Base:', base, '% → Current:', current, '%');
      if (current < base - 1) {
        console.error('Coverage decreased by more than 1%');
        process.exit(1);
      }
    "
```

## Coverage Metrics Beyond Lines

```typescript
// Branch coverage — ensures all conditional paths tested
if (user.role === 'admin') {    // branch 1
  grantFullAccess(user);
} else if (user.role === 'mod') { // branch 2
  grantModAccess(user);
} else {                          // branch 3 (often missed)
  grantReadOnly(user);
}

// Function coverage — ensures all exported functions tested
// Path coverage — ensures all execution paths through function tested
// Statement coverage — basic metric, each line executed

// Practical approach: prioritize branch coverage
// It catches the most bugs per test written
```

## Test Effectiveness Metrics

```typescript
// Track test effectiveness over time
interface TestMetrics {
  // How many bugs were caught by tests vs production?
  defectEscapeRate: number; // target: < 10%

  // How long do tests take?
  testSuiteTime: number; // target: < 5min for unit, < 15min for all

  // How often do tests fail for non-code reasons?
  flakyTestRate: number; // target: < 1%

  // What percentage of tests are useful?
  // Measure: delete test, does mutation testing still pass?
  redundantTestRate: number; // target: < 5%
}
```

## Coverage Gap Analysis

```typescript
// Identify untested critical paths
// 1. Generate coverage report
// npx vitest --coverage

// 2. Focus on uncovered branches in critical modules
// Look for:
// - Error handling paths
// - Edge cases in business logic
// - Security-related branches
// - Race condition handling

// 3. Prioritize by risk, not by line count
// Example: 10 uncovered lines in PaymentService > 100 uncovered in UI helper
```
