---
name: FastAPI Testing
description: This skill should be used when the user asks about "FastAPI test", "TestClient", "test FastAPI endpoint", or "dependency override". It covers synchronous testing with TestClient, async testing with httpx.AsyncClient, pytest fixtures and configuration, dependency overrides, database test isolation, and testing of WebSocket and background task functionality. Use this skill for any task involving test design, mocking strategies, or test infrastructure for FastAPI applications.
---

### TestClient for Synchronous Tests

To write basic synchronous tests using Starlette's TestClient (which wraps httpx):

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health_check():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

def test_create_user():
    response = client.post(
        "/api/v1/users/",
        json={"email": "test@example.com", "full_name": "Test User", "password": "Str0ng!Pass"},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert "id" in data
    assert "password" not in data  # Verify password is not exposed
```

TestClient runs the ASGI app synchronously. It is suitable for most endpoint tests but cannot test truly async behavior like concurrent request handling.

### Async Testing with httpx.AsyncClient

To test async endpoints or when the test itself needs to perform async operations, use httpx.AsyncClient with pytest-asyncio:

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.fixture
async def async_client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client

@pytest.mark.anyio
async def test_list_users(async_client: AsyncClient):
    response = await async_client.get("/api/v1/users/")
    assert response.status_code == 200
    assert isinstance(response.json(), list)

@pytest.mark.anyio
async def test_concurrent_requests(async_client: AsyncClient):
    import asyncio
    # Test that concurrent requests are handled correctly
    tasks = [
        async_client.get(f"/api/v1/users/{i}")
        for i in range(1, 6)
    ]
    responses = await asyncio.gather(*tasks)
    for resp in responses:
        assert resp.status_code in (200, 404)
```

### pytest Configuration

To configure pytest for a FastAPI project, create a `conftest.py` at the project root and a `pyproject.toml` section:

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests requiring external services",
]
filterwarnings = [
    "ignore::DeprecationWarning",
]
```

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from app.main import app
from app.dependencies import get_db
from app.models.base import Base

TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(engine, expire_on_commit=False)


@pytest.fixture(scope="session", autouse=True)
async def setup_database():
    """Create tables once for the entire test session."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture
async def db_session() -> AsyncSession:
    """Provide a transactional database session that rolls back after each test."""
    async with TestSessionLocal() as session:
        async with session.begin():
            yield session
            await session.rollback()


@pytest.fixture
async def client(db_session: AsyncSession) -> AsyncClient:
    """HTTP client with database dependency overridden."""
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

### Dependency Overrides

To replace dependencies in tests without modifying application code, use `app.dependency_overrides`:

```python
from app.dependencies import get_current_user, get_email_sender
from app.models.user import User

# Override authentication for tests
def override_get_current_user() -> User:
    return User(
        id=1,
        email="test@example.com",
        full_name="Test User",
        is_active=True,
        role="admin",
    )

@pytest.fixture(autouse=True)
def override_dependencies():
    app.dependency_overrides[get_current_user] = override_get_current_user
    yield
    app.dependency_overrides.clear()
```

To override with parameterized values for different test scenarios:

```python
def make_user_override(role: str = "member", is_active: bool = True):
    def override():
        return User(
            id=1,
            email="test@example.com",
            full_name="Test User",
            is_active=is_active,
            role=role,
        )
    return override

def test_admin_access(client):
    app.dependency_overrides[get_current_user] = make_user_override(role="admin")
    response = client.get("/admin/dashboard")
    assert response.status_code == 200

def test_member_denied_admin(client):
    app.dependency_overrides[get_current_user] = make_user_override(role="member")
    response = client.get("/admin/dashboard")
    assert response.status_code == 403
```

### Database Testing Strategies

To isolate database state between tests using transactional rollback:

```python
@pytest.fixture
async def db_session():
    """Each test runs inside a transaction that rolls back."""
    async with engine.connect() as conn:
        transaction = await conn.begin()
        session = AsyncSession(bind=conn, expire_on_commit=False)

        # Nested transaction for savepoint support
        nested = await conn.begin_nested()

        yield session

        # Rollback everything
        if nested.is_active:
            await nested.rollback()
        await transaction.rollback()
        await session.close()
```

To use factory fixtures for creating test data:

```python
@pytest.fixture
def user_factory(db_session: AsyncSession):
    async def create_user(
        email: str = "user@example.com",
        full_name: str = "Test User",
        is_active: bool = True,
        role: str = "member",
    ) -> User:
        user = User(
            email=email,
            full_name=full_name,
            hashed_password=hash_password("TestPass123!"),
            is_active=is_active,
            role=role,
        )
        db_session.add(user)
        await db_session.flush()
        return user

    return create_user

@pytest.mark.anyio
async def test_deactivate_user(client, user_factory):
    user = await user_factory(email="deactivate@example.com")
    response = await client.patch(
        f"/api/v1/users/{user.id}",
        json={"is_active": False},
    )
    assert response.status_code == 200
    assert response.json()["is_active"] is False
```

### Testing with Authentication

To test authenticated endpoints by generating real tokens in tests:

```python
from app.auth import create_access_token

@pytest.fixture
def auth_headers(db_session) -> dict[str, str]:
    token = create_access_token(data={"sub": "1", "role": "admin"})
    return {"Authorization": f"Bearer {token}"}

@pytest.mark.anyio
async def test_protected_endpoint(client, auth_headers):
    response = await client.get("/api/v1/me", headers=auth_headers)
    assert response.status_code == 200
```

### Testing Error Responses

To verify error handling produces correct response structures:

```python
@pytest.mark.anyio
async def test_not_found_returns_structured_error(client):
    response = await client.get("/api/v1/users/99999")
    assert response.status_code == 404
    data = response.json()
    assert data["error"] == "NOT_FOUND"
    assert "detail" in data

@pytest.mark.anyio
async def test_validation_error_format(client):
    response = await client.post(
        "/api/v1/users/",
        json={"email": "not-an-email", "full_name": "", "password": "weak"},
    )
    assert response.status_code == 422
    data = response.json()
    assert data["error"] == "VALIDATION_ERROR"
    assert len(data["errors"]) > 0
    assert all("field" in e and "message" in e for e in data["errors"])
```

### Test Organization

To structure tests for a FastAPI application, mirror the application structure:

```
tests/
    conftest.py              # Shared fixtures
    test_health.py           # Health check tests
    api/
        conftest.py          # API-specific fixtures
        test_users.py        # User endpoint tests
        test_products.py     # Product endpoint tests
        test_orders.py       # Order endpoint tests
    services/
        test_user_service.py # Unit tests for services
    repositories/
        test_user_repo.py    # Repository integration tests
    websockets/
        test_chat.py         # WebSocket tests
```

To run specific test categories:

```bash
# All tests
pytest

# Only API tests
pytest tests/api/

# Only fast unit tests
pytest -m "not slow and not integration"

# With coverage
pytest --cov=app --cov-report=term-missing
```

## References

- [references/testing-patterns.md](references/testing-patterns.md) - Endpoint testing, dependency mocking, async test patterns, factory fixtures, WebSocket testing, and background task testing
- [references/testing-tools.md](references/testing-tools.md) - pytest-asyncio configuration, HTTPX usage, database fixtures, test isolation, Testcontainers, and coverage reporting
