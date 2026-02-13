# Dependency Management and Supply Chain Security

Comprehensive reference covering pip-audit, safety, CVE analysis, lock file strategies, automated updates, supply chain attack patterns, private indexes, and reproducible builds.

---

## pip-audit

### Installation and Usage

```bash
pip install pip-audit

# Audit installed packages in current environment
pip-audit

# Audit a requirements file
pip-audit -r requirements.txt

# Strict mode — fail on any vulnerability
pip-audit --strict

# Output formats for CI integration
pip-audit --format json --output audit.json
pip-audit --format cyclonedx-json --output sbom.json

# Auto-fix (upgrades to patched versions)
pip-audit --fix

# Ignore specific vulnerabilities
pip-audit --ignore-vuln PYSEC-2023-123
```

### GitHub Actions Integration

```yaml
name: Security Audit
on: [push, pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - uses: pypa/gh-action-pip-audit@v1.1.0
        with:
          inputs: requirements.txt
```

---

## Safety

### Usage

```bash
pip install safety

# Check current environment
safety check

# Check a requirements file
safety check -r requirements.txt

# JSON output
safety check --json

# Only show critical/high vulnerabilities
safety check --severity-threshold high
```

---

## Understanding CVE Reports

### Severity Levels (CVSS v3)

| Score | Severity | Action |
|-------|----------|--------|
| 9.0-10.0 | Critical | Patch immediately |
| 7.0-8.9 | High | Patch within days |
| 4.0-6.9 | Medium | Patch in next sprint |
| 0.1-3.9 | Low | Evaluate and plan |

### Reading a Vulnerability Report

```
Name: requests
Version: 2.28.0
ID: PYSEC-2023-74
Fix: >=2.31.0
Description: Unintended leak of Proxy-Authorization header in redirects...
```

Key fields:
- **ID**: Unique identifier (CVE-YYYY-NNNNN or PYSEC-YYYY-NNN)
- **Fix**: Minimum version that patches the vulnerability
- **Description**: What the vulnerability enables

---

## Lock File Strategies

### pip-tools

```bash
pip install pip-tools

# Define abstract dependencies
# requirements.in
django>=5.0,<6.0
djangorestframework>=3.15
celery>=5.3

# Compile to pinned lock file
pip-compile requirements.in --output-file requirements.txt

# Upgrade all packages
pip-compile --upgrade requirements.in

# Install from lock file
pip-sync requirements.txt
```

### Poetry

```bash
# Initialize
poetry init

# Add dependencies
poetry add django djangorestframework

# Lock (generates poetry.lock)
poetry lock

# Install from lock file
poetry install --no-root
```

### uv (Recommended for Speed)

```bash
pip install uv

# Create lock file
uv lock

# Install from lock file
uv sync

# Add dependencies
uv add django
```

---

## Automated Dependency Updates

### Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: pip
    directory: "/"
    schedule:
      interval: weekly
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
    open-pull-requests-limit: 10
    groups:
      production:
        patterns:
          - "django*"
          - "celery*"
      development:
        patterns:
          - "pytest*"
          - "ruff"
          - "mypy"
```

### Renovate

```json
{
  "extends": ["config:recommended", ":pinAllExceptPeerDependencies"],
  "pip_requirements": {
    "enabled": true,
    "fileMatch": ["requirements.*\\.txt$"]
  },
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  },
  "schedule": ["every weekend"]
}
```

---

## Supply Chain Attack Patterns

### Typosquatting

Attackers publish packages with names similar to popular libraries:

```
requests → requestes, request, requesrs
django → djnago, djanogo
numpy → numppy, numby
```

**Defense**: Always verify package names. Use `pip install --dry-run` to review before installing.

### Dependency Confusion

When a company has internal packages with the same name as public packages, pip may install the public (malicious) version:

**Defense**:
```bash
# Pin your internal registry and use --index-url (not --extra-index-url)
pip install --index-url https://internal.pypi.company.com/simple/ my-internal-package

# Or use per-project pip.conf
# pip.conf
[global]
index-url = https://internal.pypi.company.com/simple/
extra-index-url = https://pypi.org/simple/
```

### Malicious Updates to Existing Packages

Maintainer account compromise or malicious code in a minor version update.

**Defense**: Pin to exact versions and review dependency diffs:
```bash
# Pin exact versions
pip-compile --generate-hashes requirements.in

# Review changes in lock file during PR review
git diff requirements.txt
```

---

## Private Package Index

### Configuring pip

```ini
# pip.conf or pip.ini
[global]
index-url = https://internal.pypi.company.com/simple/
extra-index-url = https://pypi.org/simple/
trusted-host = internal.pypi.company.com
```

### Per-Project Configuration

```toml
# pyproject.toml (Poetry)
[[tool.poetry.source]]
name = "internal"
url = "https://internal.pypi.company.com/simple/"
priority = "primary"

[[tool.poetry.source]]
name = "pypi"
url = "https://pypi.org/simple/"
priority = "supplemental"
```

---

## Reproducible Builds

### Hash-Checked Installations

```bash
# Generate requirements with hashes
pip-compile --generate-hashes requirements.in

# Output example:
# django==5.1.2 \
#     --hash=sha256:abc123... \
#     --hash=sha256:def456...

# Install with hash verification (fails if hash doesn't match)
pip install --require-hashes -r requirements.txt
```

### Verifying in CI

```yaml
- name: Install with hash verification
  run: pip install --require-hashes --no-deps -r requirements.txt
```

This ensures that the exact same package bytes are installed every time, preventing tampering between compile and install.

---

## Monitoring for New Vulnerabilities

### GitHub Security Advisories

Enable GitHub's security features:
1. **Dependabot alerts** — automatic notification of known vulnerabilities
2. **Dependabot security updates** — automatic PRs to fix vulnerabilities
3. **Secret scanning** — detect accidentally committed secrets

### OSV (Open Source Vulnerabilities)

```bash
# Install osv-scanner
pip install osv

# Scan a requirements file
osv-scanner -r requirements.txt

# Scan a lock file
osv-scanner --lockfile poetry.lock
```

### Scheduled Audits

```yaml
# .github/workflows/security-audit.yml
name: Scheduled Security Audit
on:
  schedule:
    - cron: '0 8 * * 1'  # Every Monday at 8 AM

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: |
          pip install pip-audit
          pip-audit --format json --output audit.json
      - uses: actions/upload-artifact@v4
        with:
          name: audit-report
          path: audit.json
```
