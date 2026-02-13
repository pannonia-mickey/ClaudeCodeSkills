# Testing Patterns

Comprehensive reference covering the AAA pattern, fixture composition strategies, conftest organization, test isolation techniques, integration testing, and async test patterns.

---

## Arrange-Act-Assert (AAA) Pattern

Every test should follow three clearly separated phases. Use blank lines or comments to delineate them.

### Basic AAA Structure

```python
def test_user_full_name() -> None:
    # Arrange
    user = User(first_name="Alice", last_name="Smith")

    # Act
    full_name = user.full_name()

    # Assert
    assert full_name == "Alice Smith"
```

### AAA with Fixtures

```python
def test_order_total_with_discount(
    order_factory: Callable[..., Order],
    discount_service: DiscountService,
) -> None:
    # Arrange
    order = order_factory(
        items=[
            OrderItem(product="Widget", price=10.00, quantity=5),
            OrderItem(product="Gadget", price=25.00, quantity=2),
        ]
    )

    # Act
    total = discount_service.apply_discount(order, code="SAVE10")

    # Assert
    assert total == pytest.approx(90.00)  # 100 - 10%
    assert order.discount_applied is True
```

### AAA for Exception Cases

```python
def test_withdraw_insufficient_funds() -> None:
    # Arrange
    account = BankAccount(balance=100.00)

    # Act & Assert (combined when testing exceptions)
    with pytest.raises(InsufficientFundsError) as exc_info:
        account.withdraw(150.00)

    assert exc_info.value.balance == 100.00
    assert exc_info.value.amount == 150.00
```

### Anti-Patterns to Avoid

**Multiple Act phases** -- split into separate tests:

```python
# BAD: Two actions in one test
def test_user_lifecycle() -> None:
    user = create_user("Alice")
    assert user.is_active  # First assertion block

    deactivate_user(user)
    assert not user.is_active  # Second assertion block

# GOOD: Separate tests
def test_new_user_is_active() -> None:
    user = create_user("Alice")
    assert user.is_active

def test_deactivated_user_is_inactive() -> None:
    user = create_user("Alice")
    deactivate_user(user)
    assert not user.is_active
```

**Assert-less tests** -- every test must verify something:

```python
# BAD: No assertion
def test_process_data() -> None:
    process(data)  # What is being verified?

# GOOD: Explicit assertion
def test_process_data_returns_cleaned_records() -> None:
    result = process(data)
    assert all(record.is_valid for record in result)
```

---

## Fixture Composition

### Factory Fixtures

Build flexible test data with factory functions:

```python
@pytest.fixture
def make_user(db: Database) -> Callable[..., User]:
    """Factory for creating users with sensible defaults."""
    _counter = 0

    def _make(
        name: str | None = None,
        email: str | None = None,
        role: str = "member",
    ) -> User:
        nonlocal _counter
        _counter += 1
        user = User(
            name=name or f"User {_counter}",
            email=email or f"user{_counter}@test.com",
            role=role,
        )
        db.save(user)
        return user

    return _make
```

### Layered Fixtures

Compose complex scenarios from simple building blocks:

```python
@pytest.fixture
def organization(make_user: Callable[..., User]) -> Organization:
    admin = make_user(name="Admin", role="admin")
    org = Organization(name="TestCorp", owner=admin)
    return org

@pytest.fixture
def team(organization: Organization, make_user: Callable[..., User]) -> Team:
    members = [make_user(name=f"Dev {i}") for i in range(3)]
    team = Team(name="Backend", org=organization, members=members)
    return team

@pytest.fixture
def project_with_team(team: Team) -> Project:
    return Project(name="API", team=team, status="active")
```

### Parametrized Fixtures

```python
@pytest.fixture(params=["sqlite", "postgres"])
def database(request: pytest.FixtureRequest, tmp_path: Path) -> Generator[Database, None, None]:
    if request.param == "sqlite":
        db = SQLiteDatabase(tmp_path / "test.db")
    else:
        db = PostgresDatabase(os.environ["TEST_POSTGRES_URL"])

    db.migrate()
    yield db
    db.cleanup()
```

### Fixture Teardown Patterns

```python
# Generator-based teardown (preferred)
@pytest.fixture
def temp_config(tmp_path: Path) -> Generator[Path, None, None]:
    config = tmp_path / "config.toml"
    config.write_text('[app]\ndebug = true\n')
    yield config
    # Teardown: tmp_path is auto-cleaned, but custom cleanup goes here

# addfinalizer for conditional teardown
@pytest.fixture
def server(request: pytest.FixtureRequest) -> Server:
    srv = Server()
    srv.start()
    request.addfinalizer(srv.stop)
    return srv
```

---

## conftest.py Organization

### Root conftest.py

Place session-level fixtures and shared configuration in the root:

```python
# tests/conftest.py
import pytest
from pathlib import Path

@pytest.fixture(scope="session")
def test_data_dir() -> Path:
    return Path(__file__).parent / "data"

@pytest.fixture(scope="session")
def app_config() -> dict[str, object]:
    return {
        "database_url": "sqlite:///:memory:",
        "debug": True,
        "secret_key": "test-secret-key",
    }

@pytest.fixture(autouse=True)
def reset_singletons() -> Generator[None, None, None]:
    """Ensure singletons are reset between tests."""
    yield
    SingletonRegistry.clear()
```

### Domain-Specific conftest.py

```python
# tests/unit/conftest.py
import pytest
from unittest.mock import Mock

@pytest.fixture
def mock_repository() -> Mock:
    repo = Mock(spec=UserRepository)
    repo.find_by_id.return_value = None
    repo.save.return_value = None
    return repo

@pytest.fixture
def mock_email_service() -> Mock:
    return Mock(spec=EmailService)
```

```python
# tests/integration/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres() -> Generator[PostgresContainer, None, None]:
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.fixture
def db_session(postgres: PostgresContainer) -> Generator[Session, None, None]:
    engine = create_engine(postgres.get_connection_url())
    with Session(engine) as session:
        session.begin_nested()
        yield session
        session.rollback()
```

---

## Test Isolation

### Database Isolation with Transactions

```python
@pytest.fixture
def db_session(engine: Engine) -> Generator[Session, None, None]:
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)

    yield session

    session.close()
    transaction.rollback()
    connection.close()
```

### File System Isolation

```python
@pytest.fixture
def isolated_fs(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> Path:
    monkeypatch.chdir(tmp_path)
    return tmp_path

def test_file_writer(isolated_fs: Path) -> None:
    write_output("result.txt", "data")
    assert (isolated_fs / "result.txt").read_text() == "data"
```

### Environment Variable Isolation

```python
@pytest.fixture
def clean_env(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.delenv("API_KEY", raising=False)
    monkeypatch.setenv("ENV", "test")
    monkeypatch.setenv("LOG_LEVEL", "DEBUG")

def test_config_from_env(clean_env: None) -> None:
    config = load_config()
    assert config.env == "test"
    assert config.api_key is None
```

### Time Isolation

```python
from unittest.mock import patch
from datetime import datetime

@pytest.fixture
def frozen_time() -> Generator[datetime, None, None]:
    fixed = datetime(2024, 6, 15, 12, 0, 0)
    with patch("myapp.utils.datetime") as mock_dt:
        mock_dt.now.return_value = fixed
        mock_dt.utcnow.return_value = fixed
        mock_dt.side_effect = lambda *a, **kw: datetime(*a, **kw)
        yield fixed
```

---

## Integration Testing

### Test Pyramid Approach

```
        /  E2E  \          Few, slow, high confidence
       /----------\
      / Integration \       Moderate count, moderate speed
     /----------------\
    /    Unit Tests     \   Many, fast, focused
   /______________________\
```

### API Integration Tests

```python
from fastapi.testclient import TestClient

@pytest.fixture
def client(app: FastAPI) -> TestClient:
    return TestClient(app)

def test_create_and_retrieve_user(client: TestClient) -> None:
    # Create
    response = client.post("/users", json={"name": "Alice", "email": "alice@test.com"})
    assert response.status_code == 201
    user_id = response.json()["id"]

    # Retrieve
    response = client.get(f"/users/{user_id}")
    assert response.status_code == 200
    assert response.json()["name"] == "Alice"
```

### External Service Integration

```python
@pytest.fixture
def payment_service(responses: RequestsMock) -> PaymentService:
    responses.add(
        responses.POST,
        "https://api.stripe.com/v1/charges",
        json={"id": "ch_test123", "status": "succeeded"},
        status=200,
    )
    return PaymentService(api_key="test_key")

def test_payment_processing(payment_service: PaymentService) -> None:
    result = payment_service.charge(amount=1000, currency="usd")
    assert result.status == "succeeded"
```

### Container-Based Integration

```python
import pytest
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="module")
def redis_url() -> Generator[str, None, None]:
    with RedisContainer() as redis:
        yield redis.get_connection_url()

def test_cache_operations(redis_url: str) -> None:
    cache = RedisCache(redis_url)
    cache.set("key", "value", ttl=60)
    assert cache.get("key") == "value"
```

---

## Async Testing Patterns

### pytest-asyncio Fixtures

```python
import pytest_asyncio
from httpx import AsyncClient

@pytest_asyncio.fixture
async def async_client(app: FastAPI) -> AsyncGenerator[AsyncClient, None]:
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.mark.asyncio
async def test_async_endpoint(async_client: AsyncClient) -> None:
    response = await async_client.get("/health")
    assert response.status_code == 200
```

### Testing Concurrent Operations

```python
@pytest.mark.asyncio
async def test_concurrent_writes() -> None:
    store = AsyncKeyValueStore()

    async def write(key: str, value: str) -> None:
        await store.set(key, value)

    async with asyncio.TaskGroup() as tg:
        for i in range(100):
            tg.create_task(write(f"key_{i}", f"value_{i}"))

    for i in range(100):
        assert await store.get(f"key_{i}") == f"value_{i}"
```

### Testing Timeouts

```python
@pytest.mark.asyncio
async def test_operation_timeout() -> None:
    with pytest.raises(asyncio.TimeoutError):
        async with asyncio.timeout(0.1):
            await slow_operation()
```

---

## Test Naming Conventions

Follow `test_<unit>_<scenario>_<expected_result>` or a natural language style:

```python
# Descriptive naming
def test_empty_cart_has_zero_total() -> None: ...
def test_expired_coupon_raises_error() -> None: ...
def test_bulk_order_applies_volume_discount() -> None: ...

# Class-based grouping
class TestShoppingCart:
    def test_add_item_increases_count(self) -> None: ...
    def test_remove_nonexistent_item_raises(self) -> None: ...
    def test_checkout_empty_cart_raises(self) -> None: ...
```
