---
name: Python Testing
description: This skill should be used when the user asks about "pytest", "Python test", "unittest mock", "parametrize", or "Python coverage". It covers test fundamentals, fixture design, parametrization, mocking strategies, coverage analysis, and property-based testing with Hypothesis.
---

## pytest Fundamentals

### Test Discovery and Naming

pytest discovers tests by looking for files matching `test_*.py` or `*_test.py`, classes prefixed with `Test`, and functions prefixed with `test_`. Follow these conventions strictly:

```python
# tests/test_user_service.py

class TestUserCreation:
    def test_creates_user_with_valid_data(self, user_service: UserService) -> None:
        user = user_service.create("Alice", "alice@example.com")
        assert user.name == "Alice"
        assert user.email == "alice@example.com"

    def test_raises_on_duplicate_email(self, user_service: UserService) -> None:
        user_service.create("Alice", "alice@example.com")
        with pytest.raises(DuplicateEmailError, match="already exists"):
            user_service.create("Bob", "alice@example.com")
```

### Assertions

Use plain `assert` statements. pytest rewrites them to provide detailed failure messages:

```python
def test_list_processing() -> None:
    result = process_items([1, 2, 3, 4, 5])
    assert len(result) == 3
    assert all(item > 2 for item in result)
    assert result == [3, 4, 5]
```

For floating-point comparisons, use `pytest.approx`:

```python
def test_calculation() -> None:
    assert calculate_ratio(1, 3) == pytest.approx(0.333, rel=1e-3)
```

### Exception Testing

```python
def test_invalid_input() -> None:
    with pytest.raises(ValueError, match=r"must be positive"):
        calculate(-1)

def test_exception_attributes() -> None:
    with pytest.raises(ValidationError) as exc_info:
        validate(bad_data)
    assert exc_info.value.field == "email"
    assert "invalid format" in str(exc_info.value)
```

### Markers

```python
import pytest

@pytest.mark.slow
def test_full_integration() -> None:
    ...

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_permissions() -> None:
    ...

@pytest.mark.xfail(reason="Known bug #1234", strict=True)
def test_known_issue() -> None:
    ...
```

Register custom markers in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks integration tests",
]
```

---

## Fixtures

### Basic Fixture Pattern

```python
import pytest
from myapp.database import Database

@pytest.fixture
def db() -> Database:
    """Create an in-memory database for testing."""
    return Database(":memory:")

@pytest.fixture
def populated_db(db: Database) -> Database:
    """Database pre-loaded with test data."""
    db.execute("INSERT INTO users (name) VALUES ('Alice')")
    db.execute("INSERT INTO users (name) VALUES ('Bob')")
    return db
```

### Fixture Scopes

```python
@pytest.fixture(scope="session")
def docker_db() -> Generator[Database, None, None]:
    """Spin up a Postgres container once for the entire test session."""
    container = start_postgres_container()
    db = Database(container.connection_url)
    yield db
    container.stop()

@pytest.fixture(scope="module")
def schema(docker_db: Database) -> Database:
    """Apply schema migrations once per test module."""
    docker_db.migrate()
    return docker_db

@pytest.fixture  # scope="function" is the default
def clean_db(schema: Database) -> Generator[Database, None, None]:
    """Each test gets a clean transaction that rolls back."""
    schema.begin()
    yield schema
    schema.rollback()
```

### Fixture Composition

Build complex test scenarios by composing smaller fixtures:

```python
@pytest.fixture
def user_factory(db: Database) -> Callable[..., User]:
    """Factory fixture for creating test users."""
    created: list[User] = []

    def _create(name: str = "Test User", email: str | None = None) -> User:
        email = email or f"{name.lower().replace(' ', '.')}@test.com"
        user = User(name=name, email=email)
        db.save(user)
        created.append(user)
        return user

    return _create

@pytest.fixture
def order_with_items(user_factory, product_factory) -> Order:
    user = user_factory("Alice")
    products = [product_factory(f"Item {i}") for i in range(3)]
    return Order(user=user, items=products)
```

### conftest.py Organization

Place shared fixtures in `conftest.py` at the appropriate directory level:

```
tests/
    conftest.py              # Session-scoped fixtures, shared helpers
    unit/
        conftest.py          # Unit test fixtures (mocks, fakes)
        test_models.py
        test_services.py
    integration/
        conftest.py          # Integration fixtures (database, containers)
        test_api.py
        test_workflows.py
```

---

## Parametrize

### Basic Parametrization

```python
@pytest.mark.parametrize("input_val, expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
    ("", ""),
    ("already UPPER", "ALREADY UPPER"),
])
def test_uppercase(input_val: str, expected: str) -> None:
    assert to_uppercase(input_val) == expected
```

### Parametrize with IDs

```python
@pytest.mark.parametrize("status_code, should_retry", [
    pytest.param(200, False, id="success-no-retry"),
    pytest.param(429, True, id="rate-limited-retry"),
    pytest.param(500, True, id="server-error-retry"),
    pytest.param(404, False, id="not-found-no-retry"),
])
def test_retry_logic(status_code: int, should_retry: bool) -> None:
    assert should_retry_request(status_code) == should_retry
```

### Stacking Parametrize (Cartesian Product)

```python
@pytest.mark.parametrize("method", ["GET", "POST", "PUT"])
@pytest.mark.parametrize("auth", ["token", "basic", None])
def test_endpoint_auth(method: str, auth: str | None) -> None:
    # Runs 9 test combinations (3 methods x 3 auth types)
    response = make_request(method=method, auth=auth)
    if auth is None:
        assert response.status == 401
    else:
        assert response.status != 401
```

### Indirect Parametrize

```python
@pytest.fixture
def database_engine(request: pytest.FixtureRequest) -> Engine:
    if request.param == "sqlite":
        return create_sqlite_engine()
    elif request.param == "postgres":
        return create_postgres_engine()
    raise ValueError(f"Unknown engine: {request.param}")

@pytest.mark.parametrize("database_engine", ["sqlite", "postgres"], indirect=True)
def test_query(database_engine: Engine) -> None:
    result = database_engine.execute("SELECT 1")
    assert result is not None
```

---

## Mocking

### unittest.mock Essentials

```python
from unittest.mock import Mock, MagicMock, patch, AsyncMock

def test_service_calls_repository() -> None:
    repo = Mock(spec=UserRepository)
    repo.find_by_email.return_value = User(name="Alice", email="alice@test.com")

    service = UserService(repository=repo)
    user = service.get_by_email("alice@test.com")

    repo.find_by_email.assert_called_once_with("alice@test.com")
    assert user.name == "Alice"
```

### Patching

```python
@patch("myapp.services.send_email")
def test_user_creation_sends_email(mock_send: Mock) -> None:
    service = UserService()
    service.create("Alice", "alice@test.com")

    mock_send.assert_called_once_with(
        to="alice@test.com",
        subject="Welcome!",
    )

# Context manager form
def test_with_context_manager() -> None:
    with patch("myapp.services.datetime") as mock_dt:
        mock_dt.now.return_value = datetime(2024, 1, 15)
        result = get_current_greeting()
        assert "January" in result
```

### Async Mocking

```python
async def test_async_service() -> None:
    client = AsyncMock(spec=HTTPClient)
    client.get.return_value = Response(status=200, body='{"ok": true}')

    service = APIService(client=client)
    result = await service.fetch_data("/endpoint")

    client.get.assert_awaited_once_with("/endpoint")
    assert result["ok"] is True
```

### When NOT to Mock

Avoid mocking when:
- The collaborator is a simple value object or data structure
- The mock setup is more complex than the real implementation
- Testing interaction order rather than outcomes (fragile tests)
- The collaborator is part of the same module under test

Prefer fakes (in-memory implementations) over mocks for repositories and external services in integration tests.

---

## Coverage

### Configuration

```toml
# pyproject.toml
[tool.coverage.run]
source = ["src"]
branch = true
omit = ["*/migrations/*", "*/tests/*"]

[tool.coverage.report]
fail_under = 80
show_missing = true
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "if __name__ == .__main__.",
    "@overload",
]
```

### Running Coverage

```bash
# Basic coverage run
pytest --cov=src --cov-report=term-missing

# HTML report for detailed analysis
pytest --cov=src --cov-report=html

# Combined with branch coverage
pytest --cov=src --cov-branch --cov-report=term-missing
```

### Coverage Pragmatics

Target 80-90% line coverage as a baseline. 100% coverage is rarely worthwhile because:
- Some branches are genuinely unreachable (defensive programming)
- `if TYPE_CHECKING` blocks should be excluded
- Abstract methods and `@overload` signatures do not need coverage

Focus on testing behavior, not achieving a coverage number.

---

## Hypothesis (Property-Based Testing)

### Basic Strategy Usage

```python
from hypothesis import given, settings, assume
from hypothesis import strategies as st

@given(st.lists(st.integers(), min_size=1))
def test_sort_preserves_length(xs: list[int]) -> None:
    assert len(sorted(xs)) == len(xs)

@given(st.text(min_size=1, max_size=100))
def test_encode_decode_roundtrip(text: str) -> None:
    encoded = encode(text)
    decoded = decode(encoded)
    assert decoded == text
```

### Composite Strategies

```python
from hypothesis import strategies as st

@st.composite
def user_strategy(draw: st.DrawFn) -> User:
    name = draw(st.text(min_size=1, max_size=50, alphabet=st.characters(whitelist_categories=("L",))))
    email = draw(st.emails())
    age = draw(st.integers(min_value=0, max_value=150))
    return User(name=name, email=email, age=age)

@given(user_strategy())
def test_user_serialization_roundtrip(user: User) -> None:
    data = user.to_dict()
    restored = User.from_dict(data)
    assert restored == user
```

### Settings and Profiles

```python
from hypothesis import settings, HealthCheck

@settings(max_examples=500, deadline=None)
@given(st.binary(min_size=1))
def test_compression_roundtrip(data: bytes) -> None:
    compressed = compress(data)
    assert decompress(compressed) == data

# Register profiles in conftest.py
settings.register_profile("ci", max_examples=1000)
settings.register_profile("dev", max_examples=50)
settings.load_profile(os.getenv("HYPOTHESIS_PROFILE", "dev"))
```

---

## Async Testing

### pytest-asyncio

```python
import pytest

@pytest.mark.asyncio
async def test_async_fetch() -> None:
    result = await fetch_data("https://api.example.com/data")
    assert result.status == 200

@pytest.fixture
async def async_client() -> AsyncGenerator[AsyncClient, None]:
    async with AsyncClient(base_url="http://test") as client:
        yield client

@pytest.mark.asyncio
async def test_with_async_fixture(async_client: AsyncClient) -> None:
    response = await async_client.get("/health")
    assert response.status_code == 200
```

Configure async mode in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

---

## References

- [Testing Patterns](references/testing-patterns.md) -- AAA pattern, fixture composition, conftest organization, test isolation, and integration testing strategies.
- [Testing Tools](references/testing-tools.md) -- pytest plugins, Hypothesis strategies, freezegun, responses/VCR.py, tox/nox, and mutation testing with mutmut.
