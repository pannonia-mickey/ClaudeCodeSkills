---
name: Python Packaging
description: This skill should be used when the user asks about "Python package", "pyproject.toml", "Poetry", "setuptools", or "Python project structure". It covers structuring a Python project, configuring pyproject.toml, managing dependencies, setting up virtual environments, choosing between Poetry and setuptools, and preparing a package for distribution on PyPI.
---

## pyproject.toml Configuration

`pyproject.toml` is the single configuration file for modern Python projects (PEP 621, PEP 660). Place it at the project root.

### Minimal Library Configuration (setuptools)

```toml
[build-system]
requires = ["setuptools>=69.0", "setuptools-scm>=8.0"]
build-backend = "setuptools.build_meta"

[project]
name = "my-library"
dynamic = ["version"]
description = "A short description of the library"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.10"
authors = [
    {name = "Your Name", email = "you@example.com"},
]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Typing :: Typed",
]
dependencies = [
    "httpx>=0.25.0",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=4.0",
    "mypy>=1.8",
    "ruff>=0.3",
]
docs = [
    "mkdocs>=1.5",
    "mkdocs-material>=9.0",
]

[project.urls]
Homepage = "https://github.com/org/my-library"
Documentation = "https://my-library.readthedocs.io"
Repository = "https://github.com/org/my-library"

[project.scripts]
my-cli = "my_library.cli:main"

[tool.setuptools-scm]
```

### Minimal Application Configuration (Poetry)

```toml
[tool.poetry]
name = "my-app"
version = "1.0.0"
description = "A Python application"
authors = ["Your Name <you@example.com>"]
readme = "README.md"
packages = [{include = "my_app", from = "src"}]

[tool.poetry.dependencies]
python = "^3.10"
fastapi = "^0.110"
uvicorn = {version = "^0.29", extras = ["standard"]}
sqlalchemy = "^2.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
pytest-cov = "^4.0"
mypy = "^1.8"
ruff = "^0.3"

[tool.poetry.scripts]
my-app = "my_app.main:cli"

[build-system]
requires = ["poetry-core>=1.9.0"]
build-backend = "poetry.core.masonry.api"
```

### Tool Configuration Sections

Consolidate all tool configuration in `pyproject.toml`:

```toml
[tool.ruff]
target-version = "py312"
line-length = 88
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "A", "SIM", "TCH", "RUF"]
ignore = ["E501"]

[tool.ruff.lint.isort]
known-first-party = ["my_library"]

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config"
markers = [
    "slow: marks tests as slow",
    "integration: marks integration tests",
]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
fail_under = 80
show_missing = true
```

---

## Poetry vs setuptools

### When to Use Poetry

- **Applications** (web services, CLI tools, scripts) where a lockfile is critical
- Teams that want a single tool for dependency management, virtual environments, and building
- Projects that need dependency groups (dev, test, docs)

Key commands:

```bash
# Create new project
poetry new my-project --src

# Add dependencies
poetry add httpx
poetry add --group dev pytest mypy ruff

# Install from lockfile (reproducible)
poetry install

# Update dependencies
poetry update

# Run commands in the virtual environment
poetry run pytest
poetry run mypy src

# Export to requirements.txt (for Docker, CI)
poetry export -f requirements.txt --output requirements.txt
```

### When to Use setuptools

- **Libraries** that will be published to PyPI
- Projects that need maximum compatibility with the Python ecosystem
- When depending on `setuptools-scm` for version management from git tags

Key commands:

```bash
# Install in development mode
pip install -e ".[dev]"

# Build distribution
python -m build

# Upload to PyPI
python -m twine upload dist/*
```

### When to Use Hatch

Hatch is a newer build system gaining adoption, offering environment management, versioning, and building:

```bash
# Create new project
hatch new my-project

# Run tests across Python versions
hatch test -a

# Build
hatch build
```

---

## Virtual Environments

### venv (Standard Library)

```bash
# Create
python -m venv .venv

# Activate (Unix)
source .venv/bin/activate

# Activate (Windows)
.venv\Scripts\activate

# Install dependencies
pip install -e ".[dev]"

# Deactivate
deactivate
```

### UV (Fast Package Installer)

UV is a modern, extremely fast Python package installer written in Rust:

```bash
# Install uv
pip install uv

# Create virtual environment
uv venv

# Install packages (much faster than pip)
uv pip install -e ".[dev]"

# Sync from requirements file
uv pip sync requirements.txt

# Compile lockfile
uv pip compile pyproject.toml -o requirements.txt
```

### Best Practices for Virtual Environments

- Always use a virtual environment. Never install packages globally.
- Add `.venv/` to `.gitignore`.
- Use `python -m venv` or `uv venv` for new projects.
- Pin Python version in `pyproject.toml` with `requires-python`.
- For CI, use the `actions/setup-python` GitHub Action with explicit version.

---

## Dependency Management

### Dependency Specification

```toml
# Exact (avoid for libraries, acceptable for applications)
httpx = "==0.25.0"

# Compatible release (preferred for libraries)
httpx = ">=0.25.0,<1.0"

# Poetry caret (equivalent to compatible release)
httpx = "^0.25.0"

# With extras
uvicorn = {version = ">=0.29", extras = ["standard"]}

# Platform-specific
pywin32 = {version = ">=306", markers = "sys_platform == 'win32'"}
```

### Lock Files

- **Poetry**: `poetry.lock` is generated automatically. Commit it for applications, optionally for libraries.
- **pip-tools**: Use `pip-compile` to generate `requirements.txt` from `pyproject.toml`.
- **UV**: Use `uv pip compile` for fast lock file generation.

```bash
# pip-tools workflow
pip-compile pyproject.toml -o requirements.txt
pip-compile pyproject.toml --extra dev -o requirements-dev.txt
pip install -r requirements.txt

# UV workflow
uv pip compile pyproject.toml -o requirements.txt
uv pip sync requirements.txt
```

### Dependency Groups

Organize dependencies by purpose:

```toml
[project.optional-dependencies]
dev = ["pytest", "mypy", "ruff"]
docs = ["mkdocs", "mkdocs-material"]
test = ["pytest", "pytest-cov", "hypothesis"]

# Install specific groups
# pip install -e ".[dev,test]"
```

---

## Entry Points and CLI

### Console Scripts

```toml
[project.scripts]
my-cli = "my_package.cli:main"
my-tool = "my_package.tools:run"
```

```python
# src/my_package/cli.py
import argparse

def main() -> None:
    parser = argparse.ArgumentParser(description="My CLI tool")
    parser.add_argument("--verbose", action="store_true")
    args = parser.parse_args()

    if args.verbose:
        print("Verbose mode enabled")

if __name__ == "__main__":
    main()
```

### Plugin Entry Points

```toml
[project.entry-points."my_app.plugins"]
json = "my_app.plugins.json:JSONPlugin"
yaml = "my_app.plugins.yaml:YAMLPlugin"
```

```python
# Discover plugins at runtime
from importlib.metadata import entry_points

def load_plugins() -> dict[str, type]:
    plugins = {}
    eps = entry_points(group="my_app.plugins")
    for ep in eps:
        plugins[ep.name] = ep.load()
    return plugins
```

---

## Type Checking Configuration

### py.typed Marker

For typed libraries, include a `py.typed` marker file so that type checkers recognize the package as typed:

```
src/my_library/py.typed     # Empty file
```

Declare it in `pyproject.toml`:

```toml
[tool.setuptools.package-data]
my_library = ["py.typed"]
```

### Inline vs Stub Types

- Prefer inline type annotations in the source code.
- Use stub files (`.pyi`) only when wrapping untyped third-party code or when annotations would cause circular imports.

---

## References

- [Project Structure](references/project-structure.md) -- src vs flat layout, monorepo patterns, namespace packages, entry points, and conftest.py placement.
- [Distribution Guide](references/distribution-guide.md) -- Building wheels, PyPI publishing, versioning with SemVer, dependency pinning, CI/CD pipelines, and Docker packaging.
