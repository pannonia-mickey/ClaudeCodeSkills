# Distribution Guide

Reference covering building wheels, publishing to PyPI, semantic versioning, dependency pinning strategies, CI/CD pipelines, and Docker packaging for Python projects.

---

## Building Distributions

### Wheel and sdist

Every Python package should produce both a wheel (binary distribution) and an sdist (source distribution):

```bash
# Install build tool
pip install build

# Build both wheel and sdist
python -m build

# Output in dist/
# dist/my_package-1.0.0-py3-none-any.whl
# dist/my_package-1.0.0.tar.gz
```

### Wheel Types

| Wheel Tag               | Meaning                                      |
|--------------------------|----------------------------------------------|
| `py3-none-any`          | Pure Python, any OS, any architecture         |
| `cp312-cp312-manylinux` | CPython 3.12, Linux, compiled C extensions    |
| `cp312-cp312-win_amd64` | CPython 3.12, Windows 64-bit                 |

Pure Python packages produce universal wheels. Packages with C extensions must build platform-specific wheels using tools like `cibuildwheel`.

### Building with cibuildwheel

For packages with C extensions, use `cibuildwheel` in CI to build wheels for all platforms:

```toml
# pyproject.toml
[tool.cibuildwheel]
build = "cp310-* cp311-* cp312-*"
skip = "*-musllinux_i686"
test-requires = "pytest"
test-command = "pytest {package}/tests"
```

```yaml
# .github/workflows/build.yml
jobs:
  build-wheels:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: pypa/cibuildwheel@v2
      - uses: actions/upload-artifact@v4
        with:
          path: wheelhouse/*.whl
```

---

## PyPI Publishing

### First-Time Setup

1. Create an account on [PyPI](https://pypi.org) and [Test PyPI](https://test.pypi.org).
2. Create an API token at https://pypi.org/manage/account/token/.
3. Configure credentials:

```bash
# Store token (preferred over username/password)
# In ~/.pypirc
[pypi]
  username = __token__
  password = pypi-YOUR-TOKEN-HERE

[testpypi]
  username = __token__
  password = pypi-YOUR-TEST-TOKEN-HERE
```

### Publishing Workflow

```bash
# 1. Build
python -m build

# 2. Check the distribution
python -m twine check dist/*

# 3. Upload to Test PyPI first
python -m twine upload --repository testpypi dist/*

# 4. Verify installation from Test PyPI
pip install --index-url https://test.pypi.org/simple/ my-package

# 5. Upload to production PyPI
python -m twine upload dist/*
```

### Trusted Publishing (Recommended)

Use OIDC-based trusted publishing to avoid storing API tokens. Configure in PyPI project settings to trust your GitHub repository:

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI

on:
  release:
    types: [published]

permissions:
  id-token: write  # Required for trusted publishing

jobs:
  publish:
    runs-on: ubuntu-latest
    environment: pypi
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install build
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
```

No API token configuration needed. PyPI trusts the GitHub Actions OIDC token.

---

## Versioning

### Semantic Versioning (SemVer)

Follow `MAJOR.MINOR.PATCH`:

| Component | When to Increment                                    | Example           |
|-----------|------------------------------------------------------|--------------------|
| MAJOR     | Breaking API changes                                 | 1.0.0 -> 2.0.0    |
| MINOR     | New features, backward-compatible                    | 1.0.0 -> 1.1.0    |
| PATCH     | Bug fixes, backward-compatible                       | 1.0.0 -> 1.0.1    |

Pre-release versions: `1.0.0a1` (alpha), `1.0.0b1` (beta), `1.0.0rc1` (release candidate).

### Version Management Strategies

**setuptools-scm** (derive version from git tags):

```toml
[build-system]
requires = ["setuptools>=69.0", "setuptools-scm>=8.0"]
build-backend = "setuptools.build_meta"

[project]
dynamic = ["version"]

[tool.setuptools-scm]
# Version derived from latest git tag
```

```bash
# Tag a release
git tag v1.2.0
git push --tags
# setuptools-scm reads the tag and sets the version
```

**Single source of truth in `__init__.py`**:

```python
# src/my_package/__init__.py
__version__ = "1.2.0"
```

```toml
[tool.setuptools.dynamic]
version = {attr = "my_package.__version__"}
```

**Poetry versioning**:

```bash
poetry version patch  # 1.0.0 -> 1.0.1
poetry version minor  # 1.0.0 -> 1.1.0
poetry version major  # 1.0.0 -> 2.0.0
```

### Changelog Management

Use a structured changelog format. Automate with tools like `towncrier`:

```
# CHANGELOG.md

## [1.2.0] - 2024-06-15

### Added
- New `export_csv` method on ReportService
- Support for Python 3.12

### Changed
- Upgraded httpx dependency to >=0.26

### Fixed
- Race condition in cache invalidation (#145)

### Deprecated
- `legacy_export()` function, use `export_csv()` instead
```

---

## Dependency Pinning Strategies

### For Libraries

Libraries should use loose version constraints to maximize compatibility:

```toml
[project]
dependencies = [
    "httpx>=0.25",           # Minimum version, any compatible release
    "pydantic>=2.0,<3.0",    # Range constraint
    "click>=8.0",            # Minimum only
]
```

Never pin exact versions in libraries. This causes version conflicts for downstream users.

### For Applications

Applications should pin exact versions for reproducibility:

```bash
# Generate locked requirements
pip-compile pyproject.toml -o requirements.txt --generate-hashes
uv pip compile pyproject.toml -o requirements.txt

# Or use Poetry lockfile
poetry lock
```

Example locked `requirements.txt`:

```
httpx==0.27.0 \
    --hash=sha256:abc123...
pydantic==2.6.1 \
    --hash=sha256:def456...
```

### Dependency Update Strategy

```bash
# Check for outdated packages
pip list --outdated

# Update within constraints
pip-compile --upgrade pyproject.toml -o requirements.txt

# Update a specific package
pip-compile --upgrade-package httpx pyproject.toml -o requirements.txt

# Automated updates with Dependabot or Renovate
```

### Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      dev-dependencies:
        patterns:
          - "pytest*"
          - "mypy"
          - "ruff"
```

---

## CI/CD Pipelines

### GitHub Actions: Complete Pipeline

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff
      - run: ruff check src tests
      - run: ruff format --check src tests

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - run: mypy src

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -e ".[dev,test]"
      - run: pytest --cov=src --cov-report=xml
      - uses: codecov/codecov-action@v4
        with:
          file: coverage.xml

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install pip-audit
      - run: pip-audit

  publish:
    needs: [lint, typecheck, test, security]
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    environment: pypi
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install build
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
```

### Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-yaml
      - id: check-toml
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-added-large-files
```

```bash
# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

---

## Docker Packaging

### Multi-Stage Dockerfile (Application)

```dockerfile
# Stage 1: Build
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies
RUN pip install --no-cache-dir build

# Copy only dependency files first (cache layer)
COPY pyproject.toml README.md ./
COPY src/ src/

# Build wheel
RUN python -m build --wheel

# Stage 2: Runtime
FROM python:3.12-slim AS runtime

WORKDIR /app

# Create non-root user
RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser --shell /bin/bash appuser

# Install wheel from builder
COPY --from=builder /app/dist/*.whl /tmp/
RUN pip install --no-cache-dir /tmp/*.whl && rm /tmp/*.whl

# Switch to non-root user
USER appuser

EXPOSE 8000
CMD ["uvicorn", "my_app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker with Poetry

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app

# Install Poetry
RUN pip install --no-cache-dir poetry==1.8.0

# Export to requirements.txt (avoids Poetry in runtime image)
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes

FROM python:3.12-slim AS runtime

WORKDIR /app

COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ src/

RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser appuser
USER appuser

CMD ["python", "-m", "my_app"]
```

### Docker Best Practices for Python

1. **Use slim or alpine base images** to reduce image size.
2. **Multi-stage builds** to exclude build tools from the runtime image.
3. **Copy dependency files first** to leverage Docker layer caching.
4. **Run as non-root** for security.
5. **Use `--no-cache-dir`** with pip to reduce image size.
6. **Set `PYTHONDONTWRITEBYTECODE=1`** and `PYTHONUNBUFFERED=1`:

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
```

7. **Use `.dockerignore`** to exclude unnecessary files:

```
# .dockerignore
.git
.venv
__pycache__
*.pyc
tests/
docs/
*.md
.env
.mypy_cache
.ruff_cache
```

### Docker Compose for Development

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: runtime
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

---

## Release Checklist

Follow this sequence for every release:

1. Update `CHANGELOG.md` with all changes since last release.
2. Bump the version number (via `setuptools-scm` tag, `poetry version`, or manual edit).
3. Run the full test suite: `pytest --cov=src`.
4. Run linting and type checking: `ruff check src && mypy src`.
5. Run security audit: `pip-audit`.
6. Build distributions: `python -m build`.
7. Verify the build: `twine check dist/*`.
8. Test install from the wheel: `pip install dist/*.whl`.
9. Tag the release: `git tag v1.2.0 && git push --tags`.
10. Publish: trusted publishing via GitHub Actions or `twine upload dist/*`.
11. Verify on PyPI: `pip install my-package==1.2.0`.
12. Create a GitHub Release with the changelog excerpt.
