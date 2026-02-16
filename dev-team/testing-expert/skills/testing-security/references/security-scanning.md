# Security Scanning

## Semgrep Custom Rules

```yaml
# .semgrep/custom-rules.yml
rules:
  - id: no-hardcoded-secrets
    patterns:
      - pattern: |
          $KEY = "..."
      - metavariable-regex:
          metavariable: $KEY
          regex: (password|secret|api_key|token|private_key)
    message: Do not hardcode secrets — use environment variables
    severity: ERROR
    languages: [typescript, javascript]

  - id: no-eval
    pattern: eval(...)
    message: eval() is dangerous — use safer alternatives
    severity: ERROR
    languages: [typescript, javascript]

  - id: sql-injection-risk
    patterns:
      - pattern: |
          $DB.query(`... ${$VAR} ...`)
      - pattern: |
          $DB.query("..." + $VAR + "...")
    message: Potential SQL injection — use parameterized queries
    severity: ERROR
    languages: [typescript, javascript]

  - id: no-cors-wildcard
    pattern: |
      cors({ origin: '*' })
    message: CORS wildcard allows any origin — specify allowed origins
    severity: WARNING
    languages: [typescript, javascript]

  - id: insecure-cookie
    patterns:
      - pattern: |
          { ..., secure: false, ... }
      - pattern: |
          { ..., httpOnly: false, ... }
    message: Cookies should use secure and httpOnly flags
    severity: WARNING
    languages: [typescript, javascript]
```

## Snyk Policy Configuration

```yaml
# .snyk — vulnerability policy
version: v1.25.0
ignore:
  # Ignore specific vulnerability with expiry
  SNYK-JS-LODASH-590103:
    - '*':
        reason: No user input reaches this code path
        expires: 2025-06-01T00:00:00.000Z

patch: {}

# CLI usage
# snyk test --severity-threshold=high
# snyk monitor — continuous monitoring
# snyk code test — SAST scanning
```

## Container Image Scanning

```yaml
# Trivy container scan in CI
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Trivy vulnerability scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: table
    exit-code: 1
    severity: CRITICAL,HIGH
    ignore-unfixed: true

# Dockerfile best practices scan
- name: Hadolint Dockerfile lint
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
```

## Secrets Detection

```yaml
# Pre-commit hooks with gitleaks
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

# CI scan for leaked secrets
- name: Gitleaks scan
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

```toml
# .gitleaks.toml — custom rules
[extend]
useDefault = true

[[rules]]
id = "custom-api-key"
description = "Custom API key pattern"
regex = '''(?i)my_service_key\s*[=:]\s*['"]([a-zA-Z0-9]{32,})['"]'''
secretGroup = 1

[allowlist]
paths = [
  '''\.test\.ts$''',
  '''\.spec\.ts$''',
  '''__mocks__''',
]
```

## Complete CI Security Pipeline

```yaml
# .github/workflows/security.yml
name: Security Pipeline
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 6 * * 1' # Weekly Monday scan

jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: semgrep/semgrep-action@v1
        with:
          config: p/javascript p/typescript p/owasp-top-ten .semgrep/

  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm audit --audit-level=high
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2

  dast:
    runs-on: ubuntu-latest
    needs: [sast, dependency-scan]
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - run: npm start &
      - name: Wait for app
        run: npx wait-on http://localhost:3000
      - uses: zaproxy/action-baseline@v0.13.0
        with:
          target: http://localhost:3000

  security-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx vitest --project security
```

## License Compliance

```yaml
# Check for problematic licenses
- name: License check
  run: |
    npx license-checker --failOn "GPL-2.0;GPL-3.0;AGPL-1.0;AGPL-3.0" \
      --excludePackages "my-internal-package" \
      --production
```
