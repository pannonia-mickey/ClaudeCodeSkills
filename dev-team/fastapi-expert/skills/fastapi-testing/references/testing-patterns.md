# FastAPI Testing Patterns

This reference provides comprehensive testing patterns for FastAPI applications, covering endpoint tests, dependency mocking, async testing strategies, factory fixtures, WebSocket tests, and background task verification.

---

## Endpoint Testing Patterns

### CRUD Endpoint Test Suite

A complete test suite for a resource endpoint covers all operations and edge cases:

```python
import pytest
from httpx import AsyncClient

@pytest.mark.anyio
class TestUserEndpoints:
    """Complete CRUD test suite for user endpoints."""

    async def test_create_user(self, client: AsyncClient, db_session):
        response = await client.post(
            "/api/v1/users/",
            json={
                "email": "new@example.com",
                "full_name": "New User",
                "password": "Str0ngP@ss!",
            },
        )
        assert response.status_code == 201
        data = response.json()
        assert data["email"] == "new@example.com"
        assert data["full_name"] == "New User"
        assert "id" in data
        assert "password" not in data
        assert "hashed_password" not in data

    async def test_create_user_duplicate_email(
        self, client: AsyncClient, user_factory
    ):
        await user_factory(email="existing@example.com")
        response = await client.post(
            "/api/v1/users/",
            json={
                "email": "existing@example.com",
                "full_name": "Duplicate",
                "password": "Str0ngP@ss!",
            },
        )
        assert response.status_code == 409
        assert response.json()["error"] == "CONFLICT"

    async def test_get_user(self, client: AsyncClient, user_factory):
        user = await user_factory(email="get@example.com")
        response = await client.get(f"/api/v1/users/{user.id}")
        assert response.status_code == 200
        assert response.json()["id"] == user.id

    async def test_get_user_not_found(self, client: AsyncClient):
        response = await client.get("/api/v1/users/99999")
        assert response.status_code == 404

    async def test_list_users_pagination(
        self, client: AsyncClient, user_factory
    ):
        for i in range(15):
            await user_factory(email=f"user{i}@example.com")

        # First page
        response = await client.get("/api/v1/users/?page=1&page_size=10")
        assert response.status_code == 200
        data = response.json()
        assert len(data["items"]) == 10
        assert data["total"] >= 15
        assert data["has_next"] is True

        # Second page
        response = await client.get("/api/v1/users/?page=2&page_size=10")
        data = response.json()
        assert len(data["items"]) >= 5

    async def test_update_user(self, client: AsyncClient, user_factory):
        user = await user_factory(email="update@example.com")
        response = await client.patch(
            f"/api/v1/users/{user.id}",
            json={"full_name": "Updated Name"},
        )
        assert response.status_code == 200
        assert response.json()["full_name"] == "Updated Name"
        assert response.json()["email"] == "update@example.com"  # Unchanged

    async def test_update_user_partial(self, client: AsyncClient, user_factory):
        """PATCH should only update provided fields."""
        user = await user_factory(
            email="partial@example.com", full_name="Original"
        )
        response = await client.patch(
            f"/api/v1/users/{user.id}",
            json={"full_name": "Changed"},
        )
        assert response.status_code == 200
        assert response.json()["email"] == "partial@example.com"

    async def test_delete_user(self, client: AsyncClient, user_factory):
        user = await user_factory(email="delete@example.com")
        response = await client.delete(f"/api/v1/users/{user.id}")
        assert response.status_code == 204

        # Verify deletion
        response = await client.get(f"/api/v1/users/{user.id}")
        assert response.status_code == 404
```

### Query Parameter and Filter Testing

```python
@pytest.mark.anyio
class TestProductFiltering:
    async def test_filter_by_category(self, client, product_factory):
        await product_factory(name="Widget", category_id=1)
        await product_factory(name="Gadget", category_id=2)

        response = await client.get("/api/v1/products/?category_id=1")
        data = response.json()
        assert all(p["category_id"] == 1 for p in data["items"])

    async def test_filter_by_price_range(self, client, product_factory):
        await product_factory(name="Cheap", price=5.00)
        await product_factory(name="Expensive", price=500.00)

        response = await client.get(
            "/api/v1/products/?min_price=10&max_price=100"
        )
        data = response.json()
        assert all(10 <= p["price"] <= 100 for p in data["items"])

    async def test_sort_order(self, client, product_factory):
        await product_factory(name="B Product", price=20.00)
        await product_factory(name="A Product", price=10.00)

        response = await client.get(
            "/api/v1/products/?sort_by=name&sort_order=asc"
        )
        items = response.json()["items"]
        names = [item["name"] for item in items]
        assert names == sorted(names)

    async def test_invalid_query_params(self, client):
        response = await client.get("/api/v1/products/?page=-1")
        assert response.status_code == 422
```

---

## Dependency Mocking Patterns

### Service Layer Mocking

Mock the service layer to test endpoints in isolation from business logic:

```python
from unittest.mock import AsyncMock, MagicMock
from app.services.user_service import UserService

@pytest.fixture
def mock_user_service():
    service = AsyncMock(spec=UserService)
    service.get_by_id.return_value = User(
        id=1, email="mock@example.com", full_name="Mock User"
    )
    service.create.return_value = User(
        id=2, email="new@example.com", full_name="New User"
    )
    return service

@pytest.fixture
def client_with_mock_service(mock_user_service):
    app.dependency_overrides[UserService] = lambda: mock_user_service
    transport = ASGITransport(app=app)
    with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()

@pytest.mark.anyio
async def test_get_user_calls_service(client_with_mock_service, mock_user_service):
    response = await client_with_mock_service.get("/api/v1/users/1")
    assert response.status_code == 200
    mock_user_service.get_by_id.assert_called_once_with(1)
```

### External Service Mocking

Mock external API clients injected via dependencies:

```python
@pytest.fixture
def mock_payment_gateway():
    gateway = AsyncMock()
    gateway.charge.return_value = PaymentResult(
        success=True, transaction_id="txn_123"
    )
    return gateway

@pytest.fixture
def client_with_mock_payment(mock_payment_gateway):
    app.dependency_overrides[get_payment_gateway] = lambda: mock_payment_gateway
    transport = ASGITransport(app=app)
    with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()

@pytest.mark.anyio
async def test_checkout_charges_payment(
    client_with_mock_payment, mock_payment_gateway
):
    response = await client_with_mock_payment.post(
        "/api/v1/orders/checkout",
        json={"cart_id": 1},
    )
    assert response.status_code == 201
    mock_payment_gateway.charge.assert_called_once()
```

---

## Async Test Patterns

### Testing Concurrent Operations

```python
import asyncio

@pytest.mark.anyio
async def test_concurrent_order_placement(client, product_factory, user_factory):
    """Verify that concurrent orders don't oversell stock."""
    product = await product_factory(name="Limited Item", stock=5)
    users = [await user_factory(email=f"buyer{i}@test.com") for i in range(10)]

    async def place_order(user_email: str):
        return await client.post(
            "/api/v1/orders/",
            json={"product_id": product.id, "quantity": 1},
            headers={"X-User-Email": user_email},
        )

    responses = await asyncio.gather(
        *[place_order(u.email) for u in users],
        return_exceptions=True,
    )

    successful = [r for r in responses if not isinstance(r, Exception) and r.status_code == 201]
    assert len(successful) <= 5  # Should not oversell
```

### Testing Async Event Handlers

```python
@pytest.mark.anyio
async def test_lifespan_initializes_resources():
    """Verify that lifespan startup/shutdown runs correctly."""
    from app.main import app

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        # After startup, resources should be available
        response = await client.get("/health")
        assert response.status_code == 200
        assert response.json()["checks"]["database"] == "healthy"
    # After exiting, cleanup should have run without errors
```

---

## Factory Fixture Patterns

### Composable Factories

```python
@pytest.fixture
def user_factory(db_session):
    _counter = 0

    async def create(
        email: str | None = None,
        full_name: str = "Test User",
        role: str = "member",
        is_active: bool = True,
    ) -> User:
        nonlocal _counter
        _counter += 1
        if email is None:
            email = f"user_{_counter}@test.com"

        user = User(
            email=email,
            full_name=full_name,
            hashed_password=hash_password("TestPass123!"),
            role=role,
            is_active=is_active,
        )
        db_session.add(user)
        await db_session.flush()
        return user

    return create

@pytest.fixture
def order_factory(db_session, user_factory, product_factory):
    async def create(
        user: User | None = None,
        products: list[Product] | None = None,
        status: str = "pending",
    ) -> Order:
        if user is None:
            user = await user_factory()
        if products is None:
            products = [await product_factory()]

        total = sum(p.price for p in products)
        order = Order(user_id=user.id, total=total, status=status)
        db_session.add(order)
        await db_session.flush()

        for product in products:
            item = OrderItem(
                order_id=order.id, product_id=product.id, quantity=1, price=product.price
            )
            db_session.add(item)
        await db_session.flush()

        return order

    return create
```

---

## WebSocket Testing

### Basic WebSocket Test

```python
from fastapi.testclient import TestClient

def test_websocket_connect():
    client = TestClient(app)
    with client.websocket_connect("/ws/client_1") as websocket:
        websocket.send_json({"type": "ping"})
        data = websocket.receive_json()
        assert data["type"] == "pong"

def test_websocket_broadcast():
    client = TestClient(app)
    with client.websocket_connect("/ws/client_1") as ws1:
        with client.websocket_connect("/ws/client_2") as ws2:
            ws1.send_json({"type": "message", "text": "Hello"})

            # Both clients should receive the broadcast
            data1 = ws1.receive_json()
            data2 = ws2.receive_json()
            assert data1["text"] == "Hello"
            assert data2["text"] == "Hello"
            assert data2["sender"] == "client_1"

def test_websocket_disconnect_cleanup():
    client = TestClient(app)
    with client.websocket_connect("/ws/client_1") as ws:
        ws.send_json({"type": "ping"})
        ws.receive_json()
    # After disconnect, the connection manager should have removed the client
    # Verify by checking the manager's state or attempting to send
```

### Authenticated WebSocket Testing

```python
def test_websocket_requires_auth():
    client = TestClient(app)
    # Should reject without token
    with pytest.raises(Exception):
        with client.websocket_connect("/ws/secure") as ws:
            pass

def test_websocket_with_auth():
    client = TestClient(app)
    token = create_access_token(data={"sub": "1"})
    with client.websocket_connect(
        f"/ws/secure?token={token}"
    ) as ws:
        ws.send_json({"type": "ping"})
        data = ws.receive_json()
        assert data["type"] == "pong"
```

---

## Background Task Testing

### Verifying Background Tasks Execute

```python
from unittest.mock import AsyncMock, patch

@pytest.mark.anyio
async def test_create_user_sends_welcome_email(client):
    with patch("app.routers.users.send_welcome_email", new_callable=AsyncMock) as mock_send:
        response = await client.post(
            "/api/v1/users/",
            json={
                "email": "welcome@example.com",
                "full_name": "Welcome User",
                "password": "Str0ngP@ss!",
            },
        )
        assert response.status_code == 201

        # Background task should have been scheduled
        mock_send.assert_called_once_with("welcome@example.com", "Welcome User")

@pytest.mark.anyio
async def test_background_task_failure_does_not_affect_response(client):
    """Background task failure should not cause the endpoint to fail."""
    with patch(
        "app.routers.users.send_welcome_email",
        side_effect=Exception("SMTP connection failed"),
    ):
        response = await client.post(
            "/api/v1/users/",
            json={
                "email": "bg_fail@example.com",
                "full_name": "BG Fail User",
                "password": "Str0ngP@ss!",
            },
        )
        # The response should still succeed even if the background task fails
        assert response.status_code == 201
```

### Testing Background Task Logic Independently

```python
@pytest.mark.anyio
async def test_send_welcome_email_content():
    """Test the background task function directly."""
    mock_email_client = AsyncMock()

    with patch("app.tasks.email_client", mock_email_client):
        from app.tasks import send_welcome_email
        await send_welcome_email("user@example.com", "Test User")

    mock_email_client.send.assert_called_once()
    call_kwargs = mock_email_client.send.call_args.kwargs
    assert call_kwargs["to"] == "user@example.com"
    assert call_kwargs["template"] == "welcome"
    assert call_kwargs["context"]["name"] == "Test User"
```

---

## Testing File Uploads

```python
import io

@pytest.mark.anyio
async def test_upload_avatar(client, user_factory):
    user = await user_factory()
    file_content = b"fake image content"
    response = await client.post(
        f"/api/v1/users/{user.id}/avatar",
        files={"file": ("avatar.png", io.BytesIO(file_content), "image/png")},
    )
    assert response.status_code == 200
    assert "avatar_url" in response.json()

@pytest.mark.anyio
async def test_upload_rejects_large_file(client, user_factory):
    user = await user_factory()
    large_content = b"x" * (10 * 1024 * 1024 + 1)  # Over 10MB
    response = await client.post(
        f"/api/v1/users/{user.id}/avatar",
        files={"file": ("large.png", io.BytesIO(large_content), "image/png")},
    )
    assert response.status_code == 413

@pytest.mark.anyio
async def test_upload_rejects_invalid_mime(client, user_factory):
    user = await user_factory()
    response = await client.post(
        f"/api/v1/users/{user.id}/avatar",
        files={"file": ("script.exe", io.BytesIO(b"malicious"), "application/octet-stream")},
    )
    assert response.status_code == 422
```
