# FastAPI Testing Tools

This reference covers the testing tool ecosystem for FastAPI applications, including pytest-asyncio configuration, HTTPX patterns, database test fixtures, test isolation strategies, Testcontainers for integration testing, and coverage reporting.

---

## pytest-asyncio

### Installation and Configuration

```bash
pip install pytest-asyncio
```

Configure the async mode in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

The `auto` mode automatically treats all `async def` test functions as async tests without requiring the `@pytest.mark.anyio` decorator on each one. However, explicit markers are still recommended for clarity.

### Fixture Scoping with Async

```python
import pytest

# Session-scoped async fixture - runs once for the entire test session
@pytest.fixture(scope="session")
async def database_engine():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

# Module-scoped async fixture - runs once per test module
@pytest.fixture(scope="module")
async def seed_data(database_engine):
    async with async_sessionmaker(database_engine)() as session:
        session.add_all([
            Category(name="Electronics"),
            Category(name="Books"),
            Category(name="Clothing"),
        ])
        await session.commit()
    yield

# Function-scoped async fixture (default) - runs for each test
@pytest.fixture
async def db_session(database_engine):
    async with async_sessionmaker(database_engine)() as session:
        async with session.begin():
            yield session
            await session.rollback()
```

### Event Loop Configuration

For custom event loop policies:

```python
# conftest.py
import pytest
import asyncio

@pytest.fixture(scope="session")
def event_loop_policy():
    """Use uvloop for faster async tests if available."""
    try:
        import uvloop
        return uvloop.EventLoopPolicy()
    except ImportError:
        return asyncio.DefaultEventLoopPolicy()
```

---

## HTTPX for Testing

### AsyncClient Configuration

```python
from httpx import AsyncClient, ASGITransport

@pytest.fixture
async def client(db_session) -> AsyncClient:
    """Configured async HTTP client for testing."""

    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    transport = ASGITransport(app=app)
    async with AsyncClient(
        transport=transport,
        base_url="http://test",
        headers={"Content-Type": "application/json"},
        timeout=30.0,
    ) as client:
        yield client

    app.dependency_overrides.clear()
```

### Authenticated Client Fixture

```python
@pytest.fixture
async def admin_client(client: AsyncClient, user_factory) -> AsyncClient:
    """Client authenticated as an admin user."""
    admin = await user_factory(email="admin@test.com", role="admin")
    token = create_access_token(data={"sub": str(admin.id), "role": "admin"})
    client.headers["Authorization"] = f"Bearer {token}"
    return client

@pytest.fixture
async def member_client(client: AsyncClient, user_factory) -> AsyncClient:
    """Client authenticated as a regular member."""
    member = await user_factory(email="member@test.com", role="member")
    token = create_access_token(data={"sub": str(member.id), "role": "member"})
    client.headers["Authorization"] = f"Bearer {token}"
    return client
```

### Testing File Downloads

```python
@pytest.mark.anyio
async def test_download_report(admin_client: AsyncClient):
    response = await admin_client.get("/api/v1/reports/monthly")
    assert response.status_code == 200
    assert response.headers["content-type"] == "text/csv"
    assert "content-disposition" in response.headers

    # Verify CSV content
    lines = response.text.strip().split("\n")
    assert len(lines) > 1  # Header + data rows
    header = lines[0].split(",")
    assert "date" in header
    assert "revenue" in header
```

### Testing Redirect Responses

```python
@pytest.mark.anyio
async def test_oauth_redirect(client: AsyncClient):
    response = await client.get(
        "/api/v1/auth/google",
        follow_redirects=False,
    )
    assert response.status_code == 307
    assert "accounts.google.com" in response.headers["location"]
```

---

## Database Test Fixtures

### Transaction-Based Isolation

The most efficient database isolation strategy wraps each test in a transaction that rolls back:

```python
@pytest.fixture
async def db_session(database_engine) -> AsyncSession:
    """
    Provide a transactional session per test.
    All changes are rolled back after each test.
    """
    connection = await database_engine.connect()
    transaction = await connection.begin()

    session = AsyncSession(
        bind=connection,
        join_transaction_mode="create_savepoint",
        expire_on_commit=False,
    )

    yield session

    await session.close()
    await transaction.rollback()
    await connection.close()
```

### Fresh Database Per Test Module

For tests that need a clean database (e.g., migration tests):

```python
@pytest.fixture(scope="module")
async def fresh_database():
    """Create a fresh database for each test module."""
    from sqlalchemy import text

    admin_engine = create_async_engine(ADMIN_DATABASE_URL)
    db_name = f"test_{uuid4().hex[:8]}"

    async with admin_engine.connect() as conn:
        await conn.execute(text("COMMIT"))
        await conn.execute(text(f"CREATE DATABASE {db_name}"))

    test_url = f"postgresql+asyncpg://user:pass@localhost:5432/{db_name}"
    test_engine = create_async_engine(test_url)

    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield test_engine

    await test_engine.dispose()
    async with admin_engine.connect() as conn:
        await conn.execute(text("COMMIT"))
        await conn.execute(text(f"DROP DATABASE {db_name}"))
    await admin_engine.dispose()
```

### Seeding Test Data

```python
@pytest.fixture
async def seeded_db(db_session: AsyncSession):
    """Seed the database with a standard dataset for testing."""
    categories = [
        Category(id=1, name="Electronics"),
        Category(id=2, name="Books"),
        Category(id=3, name="Clothing"),
    ]
    db_session.add_all(categories)

    products = [
        Product(id=1, name="Laptop", price=999.99, category_id=1, stock=50),
        Product(id=2, name="Python Book", price=49.99, category_id=2, stock=200),
        Product(id=3, name="T-Shirt", price=19.99, category_id=3, stock=500),
    ]
    db_session.add_all(products)
    await db_session.flush()

    return {"categories": categories, "products": products}
```

---

## Test Isolation Strategies

### Dependency Override Cleanup

Always clean up dependency overrides to prevent test pollution:

```python
@pytest.fixture(autouse=True)
def clean_dependency_overrides():
    """Reset dependency overrides after each test."""
    yield
    app.dependency_overrides.clear()
```

### Environment Variable Isolation

```python
import os
from unittest.mock import patch

@pytest.fixture
def test_environment():
    """Provide clean environment variables for testing."""
    env_overrides = {
        "APP_ENVIRONMENT": "testing",
        "APP_DEBUG": "true",
        "APP_DATABASE_URL": "sqlite+aiosqlite:///./test.db",
        "APP_SECRET_KEY": "test-secret-key-not-for-production",
    }
    with patch.dict(os.environ, env_overrides, clear=False):
        # Clear cached settings
        get_settings.cache_clear()
        yield
        get_settings.cache_clear()
```

### Freezing Time

```python
from freezegun import freeze_time

@freeze_time("2025-06-15 12:00:00")
def test_token_expiration():
    token = create_access_token(
        data={"sub": "1"}, expires_delta=timedelta(minutes=30)
    )
    payload = decode_jwt(token)
    assert payload["exp"] == 1750003800  # 12:30:00 UTC

@freeze_time("2025-06-15 13:00:00")
def test_token_is_expired():
    # Token created at noon, checking at 1 PM (expired at 12:30)
    with pytest.raises(ExpiredTokenError):
        decode_jwt(expired_token)
```

---

## Testcontainers

Testcontainers provides disposable Docker containers for integration testing with real databases and services.

### PostgreSQL Testcontainer

```bash
pip install testcontainers[postgres]
```

```python
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
async def postgres_engine():
    """Spin up a real PostgreSQL instance for integration tests."""
    with PostgresContainer("postgres:16-alpine") as postgres:
        url = postgres.get_connection_url()
        # Convert to async URL
        async_url = url.replace("psycopg2", "asyncpg")
        engine = create_async_engine(async_url)

        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)

        yield engine

        await engine.dispose()
```

### Redis Testcontainer

```python
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7-alpine") as redis:
        yield redis

@pytest.fixture
async def redis_client(redis_container):
    import aioredis
    url = f"redis://{redis_container.get_container_host_ip()}:{redis_container.get_exposed_port(6379)}"
    client = await aioredis.from_url(url)
    yield client
    await client.flushdb()
    await client.close()
```

### Compose-Based Integration Tests

```python
from testcontainers.compose import DockerCompose

@pytest.fixture(scope="session")
def compose():
    """Start the full application stack for end-to-end tests."""
    with DockerCompose(
        filepath=".",
        compose_file_name="docker-compose.test.yml",
        pull=True,
    ) as compose:
        compose.wait_for("http://localhost:8000/health")
        yield compose
```

---

## Coverage Reporting

### Configuration

```toml
# pyproject.toml
[tool.coverage.run]
source = ["app"]
omit = [
    "app/migrations/*",
    "app/config.py",
    "*/tests/*",
]
branch = true

[tool.coverage.report]
fail_under = 80
show_missing = true
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if TYPE_CHECKING:",
    "if __name__ == .__main__.",
    "raise NotImplementedError",
    "pass",
    "\\.\\.\\.",
]

[tool.coverage.html]
directory = "htmlcov"
```

### Running with Coverage

```bash
# Run tests with coverage
pytest --cov=app --cov-report=term-missing

# Generate HTML report
pytest --cov=app --cov-report=html

# Generate XML for CI integration
pytest --cov=app --cov-report=xml:coverage.xml

# Combine coverage from multiple test runs
coverage combine
coverage report
```

### CI Integration

```yaml
# .github/workflows/test.yml
- name: Run tests
  run: pytest --cov=app --cov-report=xml:coverage.xml

- name: Upload coverage
  uses: codecov/codecov-action@v4
  with:
    files: coverage.xml
    fail_ci_if_error: true
```

---

## Markers and Test Selection

### Custom Markers

```python
# conftest.py
import pytest

def pytest_configure(config):
    config.addinivalue_line("markers", "slow: marks tests as slow-running")
    config.addinivalue_line("markers", "integration: requires external services")
    config.addinivalue_line("markers", "e2e: end-to-end tests")
```

### Usage in Tests

```python
@pytest.mark.slow
@pytest.mark.anyio
async def test_large_data_export(client):
    """Test exporting 100k records - takes ~30s."""
    ...

@pytest.mark.integration
@pytest.mark.anyio
async def test_payment_processing(client):
    """Requires Stripe test account."""
    ...

@pytest.mark.e2e
def test_full_checkout_flow(compose):
    """Full end-to-end checkout against Docker Compose stack."""
    ...
```

### Running Specific Categories

```bash
# Run only fast unit tests
pytest -m "not slow and not integration and not e2e"

# Run integration tests only
pytest -m integration

# Run everything except e2e
pytest -m "not e2e"

# Run tests matching a keyword
pytest -k "test_user and not test_user_delete"
```

---

## Performance Testing Utilities

### Simple Benchmark Fixture

```python
import time

@pytest.fixture
def benchmark():
    """Simple benchmark utility for endpoint performance testing."""
    class Benchmark:
        def __init__(self):
            self.results = []

        async def run(self, coro, iterations: int = 100):
            times = []
            for _ in range(iterations):
                start = time.perf_counter()
                await coro()
                elapsed = time.perf_counter() - start
                times.append(elapsed)
            self.results = times
            return {
                "min": min(times),
                "max": max(times),
                "avg": sum(times) / len(times),
                "p95": sorted(times)[int(len(times) * 0.95)],
                "p99": sorted(times)[int(len(times) * 0.99)],
            }

    return Benchmark()

@pytest.mark.slow
@pytest.mark.anyio
async def test_list_endpoint_performance(client, benchmark, seeded_db):
    async def make_request():
        await client.get("/api/v1/products/?page=1&page_size=20")

    stats = await benchmark.run(make_request, iterations=200)
    assert stats["p95"] < 0.1  # 95th percentile under 100ms
```
