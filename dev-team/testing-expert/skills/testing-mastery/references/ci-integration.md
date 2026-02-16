# CI Test Integration

## Parallel Test Execution

```yaml
# GitHub Actions — matrix strategy for parallel test shards
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npx vitest --shard=${{ matrix.shard }}/4

  # Merge coverage reports
  coverage:
    needs: test
    steps:
      - run: npx istanbul-merge --out combined.json shard-*.json
      - run: npx istanbul report --include combined.json lcov
```

## Test Splitting by Timing

```yaml
# CircleCI-style timing-based splitting
- run: |
    TESTS=$(circleci tests glob "tests/**/*.test.ts" | circleci tests split --split-by=timings)
    npx vitest $TESTS
```

## Flaky Test Detection

```yaml
# Retry flaky tests before failing
- run: npx vitest --retry=2

# Track flaky tests over time
- run: |
    npx vitest --reporter=json > test-results.json
    # Upload results to flaky test tracker
    curl -X POST https://flaky-tracker.internal/api/results \
      -H "Content-Type: application/json" \
      -d @test-results.json
```

## Test Reporting

```yaml
# JUnit XML output for CI
- run: npx vitest --reporter=junit --outputFile=test-results.xml

# Upload test results
- uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Test Results
    path: test-results.xml
    reporter: java-junit

# Coverage reporting
- uses: codecov/codecov-action@v4
  with:
    files: coverage/lcov.info
    fail_ci_if_error: true
```

## Artifact Management

```yaml
# Save failure screenshots and traces
- uses: actions/upload-artifact@v4
  if: failure()
  with:
    name: test-artifacts
    path: |
      test-results/
      playwright-report/
    retention-days: 7
```

## Quality Gates

```yaml
# Enforce minimum coverage
- run: |
    npx vitest --coverage
    COVERAGE=$(node -e "const r=require('./coverage/coverage-summary.json'); console.log(r.total.lines.pct)")
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "Coverage $COVERAGE% is below 80% threshold"
      exit 1
    fi

# Enforce no test skips in main branch
- run: |
    SKIPPED=$(grep -c "\.skip\|xit\|xdescribe" tests/**/*.test.ts || true)
    if [ "$SKIPPED" -gt 0 ]; then
      echo "Found $SKIPPED skipped tests. Remove .skip before merging."
      exit 1
    fi
```

## Test Environment Management

```yaml
# Docker Compose for test dependencies
services:
  test:
    build: .
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
    environment:
      DATABASE_URL: postgresql://test:test@postgres:5432/test
      REDIS_URL: redis://redis:6379
  postgres:
    image: postgres:16
    healthcheck:
      test: pg_isready
      interval: 5s
  redis:
    image: redis:7
```
