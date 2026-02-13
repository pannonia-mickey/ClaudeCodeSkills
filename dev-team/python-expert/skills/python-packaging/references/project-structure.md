# Project Structure

Reference covering src vs flat layout, monorepo patterns, namespace packages, entry points, and test organization with conftest.py placement.

---

## src Layout vs Flat Layout

### src Layout (Recommended)

The `src` layout places all package source code inside a `src/` directory. This is the recommended layout for libraries and applications because it prevents accidental imports from the project root.

```
my-project/
    pyproject.toml
    README.md
    LICENSE
    src/
        my_package/
            __init__.py
            core.py
            models.py
            utils.py
            py.typed
    tests/
        __init__.py
        conftest.py
        unit/
            __init__.py
            conftest.py
            test_core.py
            test_models.py
        integration/
            __init__.py
            conftest.py
            test_api.py
    docs/
        index.md
```

Configuration for src layout with setuptools:

```toml
[tool.setuptools.packages.find]
where = ["src"]
```

Configuration for src layout with Poetry:

```toml
[tool.poetry]
packages = [{include = "my_package", from = "src"}]
```

**Benefits of src layout:**
- Forces installation before import, catching packaging errors early
- Prevents the project root from polluting `sys.path`
- Clearly separates source code from project configuration
- Tests always run against the installed package, not source files

### Flat Layout

The flat layout places the package directly in the project root. Acceptable for small scripts, single-module packages, or internal tools.

```
my-project/
    pyproject.toml
    README.md
    my_package/
        __init__.py
        core.py
    tests/
        test_core.py
```

**Drawbacks:**
- Can accidentally import from the project root instead of the installed package
- Harder to catch missing `__init__.py` or missing files in the distribution

### Single-Module Package

For very small packages that are a single file:

```
my-project/
    pyproject.toml
    README.md
    src/
        my_module.py
    tests/
        test_my_module.py
```

```toml
[tool.setuptools]
py-modules = ["my_module"]

[tool.setuptools.packages.find]
where = ["src"]
```

---

## Application Project Structure

For web applications, CLI tools, or services:

```
my-app/
    pyproject.toml
    README.md
    docker-compose.yml
    Dockerfile
    .env.example
    src/
        my_app/
            __init__.py
            main.py
            config.py
            api/
                __init__.py
                routes.py
                middleware.py
                dependencies.py
            domain/
                __init__.py
                models.py
                services.py
                repositories.py
            infrastructure/
                __init__.py
                database.py
                cache.py
                messaging.py
            cli/
                __init__.py
                commands.py
    tests/
        conftest.py
        unit/
            conftest.py
            domain/
                test_models.py
                test_services.py
        integration/
            conftest.py
            test_api.py
            test_database.py
        e2e/
            conftest.py
            test_workflows.py
    migrations/
        versions/
            001_initial.py
    scripts/
        seed_data.py
```

### Layer Organization

| Layer          | Responsibility                     | Dependencies       |
|----------------|------------------------------------|--------------------|
| `api/`         | HTTP handlers, request/response    | `domain/`          |
| `domain/`      | Business logic, models, services   | None (pure Python) |
| `infrastructure/` | Database, cache, external APIs  | `domain/`          |
| `cli/`         | Command-line interface             | `domain/`          |

The domain layer has no dependencies on infrastructure or API layers. Use dependency injection to connect them.

---

## Monorepo Patterns

### Shared-Root Monorepo

Multiple packages under a single repository root, each with its own `pyproject.toml`:

```
monorepo/
    pyproject.toml              # Root workspace (optional, for tooling)
    packages/
        core/
            pyproject.toml
            src/
                core_lib/
                    __init__.py
            tests/
                test_core.py
        api/
            pyproject.toml
            src/
                api_service/
                    __init__.py
            tests/
                test_api.py
        worker/
            pyproject.toml
            src/
                worker_service/
                    __init__.py
            tests/
                test_worker.py
    scripts/
        install_all.sh
```

### Cross-Package Dependencies

In a monorepo, packages can depend on each other via path references:

```toml
# packages/api/pyproject.toml
[project]
dependencies = [
    "core-lib",  # Published name
]

# For development, install as editable
# pip install -e ../core -e ".[dev]"
```

With Poetry workspaces (experimental):

```toml
# packages/api/pyproject.toml
[tool.poetry.dependencies]
core-lib = {path = "../core", develop = true}
```

### Monorepo Tooling

- **pants** or **bazel**: Full-featured build systems for large monorepos
- **nox/tox**: Run tests across all packages
- **GitHub Actions matrix**: Test each package independently

```yaml
# .github/workflows/test.yml
strategy:
  matrix:
    package: [core, api, worker]
steps:
  - run: pip install -e "./packages/${{ matrix.package }}[dev]"
  - run: pytest packages/${{ matrix.package }}/tests
```

---

## Namespace Packages

Namespace packages allow splitting a single logical package across multiple distributions (PEP 420).

### Implicit Namespace Packages (Recommended)

Omit `__init__.py` from the namespace directory:

```
# Distribution 1: mycompany-core
src/
    mycompany/                # No __init__.py!
        core/
            __init__.py
            models.py

# Distribution 2: mycompany-auth
src/
    mycompany/                # No __init__.py!
        auth/
            __init__.py
            tokens.py
```

```python
# After installing both packages:
from mycompany.core.models import User
from mycompany.auth.tokens import create_token
```

### Configuration for Namespace Packages

```toml
# mycompany-core/pyproject.toml
[project]
name = "mycompany-core"

[tool.setuptools.packages.find]
where = ["src"]
```

The key is that the `mycompany/` directory must NOT contain an `__init__.py` file in either distribution.

---

## Entry Points

### Console Script Entry Points

Define CLI commands that are installed with the package:

```toml
[project.scripts]
my-cli = "my_package.cli:main"
my-admin = "my_package.admin:cli"
```

```python
# src/my_package/cli.py
import argparse
import sys

def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="my-cli")
    parser.add_argument("command", choices=["run", "check", "migrate"])
    parser.add_argument("--verbose", "-v", action="store_true")

    args = parser.parse_args(argv)

    match args.command:
        case "run":
            return run_app(verbose=args.verbose)
        case "check":
            return run_checks()
        case "migrate":
            return run_migrations()
        case _:
            parser.print_help()
            return 1

if __name__ == "__main__":
    sys.exit(main())
```

### GUI Scripts

```toml
[project.gui-scripts]
my-gui-app = "my_package.gui:main"
```

### Plugin Entry Points

Enable third-party extensibility:

```toml
# In the host package
[project.entry-points."my_app.plugins"]
builtin_json = "my_app.plugins.json_handler:JSONPlugin"

# In a third-party plugin package
[project.entry-points."my_app.plugins"]
custom_xml = "third_party_plugin.xml_handler:XMLPlugin"
```

```python
# Host package: discover and load plugins
from importlib.metadata import entry_points

def discover_plugins() -> dict[str, type]:
    plugins = {}
    for ep in entry_points(group="my_app.plugins"):
        try:
            plugin_class = ep.load()
            plugins[ep.name] = plugin_class
        except Exception as e:
            print(f"Failed to load plugin {ep.name}: {e}")
    return plugins
```

---

## conftest.py Placement

### Placement Rules

1. **Root `tests/conftest.py`**: Session-scoped fixtures, pytest plugins, shared helpers available to all tests.
2. **Category `tests/unit/conftest.py`**: Fixtures specific to unit tests (mocks, fakes, factories).
3. **Category `tests/integration/conftest.py`**: Fixtures for integration tests (database containers, HTTP stubs).
4. **Domain `tests/unit/domain/conftest.py`**: Narrow fixtures for a specific domain area.

### Example Hierarchy

```python
# tests/conftest.py
import pytest
from pathlib import Path

@pytest.fixture(scope="session")
def test_data_dir() -> Path:
    """Available to ALL tests."""
    return Path(__file__).parent / "fixtures" / "data"

@pytest.fixture(scope="session")
def app_config() -> dict[str, object]:
    return {"env": "test", "debug": True}


# tests/unit/conftest.py
import pytest
from unittest.mock import Mock

@pytest.fixture
def mock_db() -> Mock:
    """Available only to unit tests."""
    return Mock(spec=Database)

@pytest.fixture
def mock_cache() -> Mock:
    return Mock(spec=CacheBackend)


# tests/integration/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres_container() -> PostgresContainer:
    """Available only to integration tests."""
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.fixture
def db_session(postgres_container):
    engine = create_engine(postgres_container.get_connection_url())
    with Session(engine) as session:
        yield session
```

### Test Data Files

Place test fixtures and sample data alongside tests:

```
tests/
    fixtures/
        data/
            sample_input.json
            expected_output.json
            large_dataset.csv
        cassettes/              # VCR.py HTTP recordings
            api_fetch.yaml
    factories/
        __init__.py
        user_factory.py
        order_factory.py
```

Access via the `test_data_dir` fixture:

```python
def test_data_processing(test_data_dir: Path) -> None:
    input_path = test_data_dir / "sample_input.json"
    data = json.loads(input_path.read_text())
    result = process(data)
    assert result.is_valid
```

---

## Package Data and Resources

### Including Non-Python Files

```toml
[tool.setuptools.package-data]
my_package = [
    "py.typed",
    "templates/*.html",
    "static/**/*",
    "data/*.json",
]
```

### Accessing Package Resources

```python
from importlib.resources import files, as_file

# Read a text resource
template = files("my_package.templates").joinpath("base.html").read_text()

# Read a binary resource
with as_file(files("my_package.data").joinpath("defaults.json")) as path:
    data = json.loads(path.read_text())
```

Use `importlib.resources` instead of `pkg_resources` (deprecated) or `__file__`-based paths (fragile in zip archives and wheels).
