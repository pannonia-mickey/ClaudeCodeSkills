# FastAPI Anti-Patterns Catalog

This reference catalogs common anti-patterns found in FastAPI applications with concrete before/after refactoring examples. Each pattern includes why it is problematic and how to fix it.

---

## Sync Blocking in Async Endpoints

### Problem

Calling synchronous blocking functions inside `async def` endpoints blocks the event loop, preventing all other requests from being processed concurrently.

```python
# BAD: Blocking call inside async endpoint
@router.post("/reports")
async def generate_report(params: ReportParams, db: AsyncSession = Depends(get_db)):
    data = await db.execute(select(SalesData).where(...))
    rows = data.scalars().all()

    # This blocks the event loop for seconds
    pdf_bytes = generate_pdf_report(rows)  # CPU-bound, synchronous

    return Response(content=pdf_bytes, media_type="application/pdf")
```

### Fix

Use `def` instead of `async def` for CPU-bound handlers (FastAPI runs them in a threadpool), or offload the work explicitly:

```python
# GOOD Option A: Use sync def - FastAPI runs it in threadpool
@router.post("/reports")
def generate_report(params: ReportParams):
    # Entire handler runs in threadpool, won't block event loop
    data = sync_db.execute(...)
    pdf_bytes = generate_pdf_report(data)
    return Response(content=pdf_bytes, media_type="application/pdf")

# GOOD Option B: Keep async, offload blocking work
import asyncio
from functools import partial

@router.post("/reports")
async def generate_report(params: ReportParams, db: AsyncSession = Depends(get_db)):
    data = await db.execute(select(SalesData).where(...))
    rows = data.scalars().all()

    # Offload CPU-bound work to threadpool
    loop = asyncio.get_event_loop()
    pdf_bytes = await loop.run_in_executor(None, partial(generate_pdf_report, rows))

    return Response(content=pdf_bytes, media_type="application/pdf")
```

---

## N+1 Query Problem

### Problem

Accessing related objects in a loop triggers one query per iteration instead of loading all relationships upfront.

```python
# BAD: N+1 queries - 1 query for orders + N queries for users
@router.get("/orders/")
async def list_orders(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Order).limit(50))
    orders = result.scalars().all()

    response = []
    for order in orders:
        # Each iteration triggers a lazy load query for user
        user = await db.execute(select(User).where(User.id == order.user_id))
        user = user.scalar_one()
        response.append({
            "id": order.id,
            "total": order.total,
            "user_name": user.full_name,  # N extra queries
        })
    return response
```

### Fix

Use eager loading to fetch related data in a single query or a minimal number of queries:

```python
# GOOD: Eager loading with selectinload
from sqlalchemy.orm import selectinload, joinedload

@router.get("/orders/", response_model=list[OrderWithUserResponse])
async def list_orders(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Order)
        .options(joinedload(Order.user))  # JOIN in same query
        .limit(50)
    )
    return result.scalars().all()

# For collections, use selectinload (separate IN query, avoids cartesian product)
result = await db.execute(
    select(Order)
    .options(selectinload(Order.items).joinedload(OrderItem.product))
    .limit(50)
)
```

---

## Overly Broad Exception Handling

### Problem

Catching `Exception` hides bugs and makes debugging impossible.

```python
# BAD: Swallowing all exceptions
@router.post("/users/")
async def create_user(user_in: UserCreate, db: AsyncSession = Depends(get_db)):
    try:
        user = User(**user_in.model_dump())
        db.add(user)
        await db.commit()
        return user
    except Exception:
        await db.rollback()
        raise HTTPException(500, "Something went wrong")  # No clue what happened
```

### Fix

Catch specific exceptions and handle them appropriately:

```python
# GOOD: Specific exception handling
from sqlalchemy.exc import IntegrityError

@router.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(user_in: UserCreate, service: UserService = Depends()):
    return await service.create(user_in)

# In the service:
class UserService:
    async def create(self, data: UserCreate) -> User:
        try:
            return await self.repo.create(
                email=data.email,
                full_name=data.full_name,
                hashed_password=hash_password(data.password),
            )
        except IntegrityError:
            raise ConflictError("User", "email", data.email)
```

---

## Hardcoded Configuration

### Problem

Embedding connection strings, API keys, or magic numbers directly in code.

```python
# BAD: Hardcoded values
engine = create_async_engine("postgresql+asyncpg://admin:secret@prod-db:5432/myapp")
SECRET_KEY = "my-secret-key-2024"
TOKEN_EXPIRY = 1800
```

### Fix

Use `pydantic-settings` for typed, validated configuration from environment variables:

```python
# GOOD: Configuration from environment
from pydantic_settings import BaseSettings
from pydantic import SecretStr

class Settings(BaseSettings):
    database_url: SecretStr
    secret_key: SecretStr
    access_token_expire_minutes: int = 30

    class Config:
        env_prefix = "APP_"
        env_file = ".env"
```

---

## Mutable Default Arguments

### Problem

Using mutable defaults in Pydantic models or function signatures leads to shared state across calls.

```python
# BAD: Mutable default shared across all calls
class OrderCreate(BaseModel):
    items: list[OrderItemCreate] = []  # All instances share this list!

# BAD: Mutable default in function
async def create_order(items: list = []):
    items.append(new_item)  # Mutates the default for all future calls
```

### Fix

Use `Field(default_factory=...)` for Pydantic or `None` with initialization for functions:

```python
# GOOD: Proper default factory
class OrderCreate(BaseModel):
    items: list[OrderItemCreate] = Field(default_factory=list)

# GOOD: None default with initialization
async def create_order(items: list | None = None):
    items = items or []
```

---

## Dependency Scope Confusion

### Problem

Using module-level state or singletons where request-scoped dependencies are needed.

```python
# BAD: Module-level session shared across requests (NOT thread-safe)
db_session = AsyncSession(engine)

@router.get("/users/")
async def list_users():
    result = await db_session.execute(select(User))  # Shared session = data leaks
    return result.scalars().all()
```

### Fix

Use `Depends()` for request-scoped resources:

```python
# GOOD: Request-scoped session via Depends()
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        yield session

@router.get("/users/", response_model=list[UserResponse])
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()
```

---

## Missing Response Model

### Problem

Omitting `response_model` allows any data (including sensitive fields) to leak through the API.

```python
# BAD: No response_model - returns everything including hashed_password
@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    return await db.get(User, user_id)
```

### Fix

Always declare `response_model` to control the API contract:

```python
# GOOD: Explicit response model filters the output
@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, service: UserService = Depends()):
    return await service.get_or_404(user_id)
```

---

## God Service

### Problem

A single service class that handles everything for a domain, growing to hundreds or thousands of lines.

```python
# BAD: God service with too many responsibilities
class UserService:
    async def create(self, data): ...
    async def update(self, id, data): ...
    async def delete(self, id): ...
    async def login(self, credentials): ...
    async def logout(self, token): ...
    async def refresh_token(self, token): ...
    async def reset_password(self, email): ...
    async def verify_email(self, token): ...
    async def update_avatar(self, id, file): ...
    async def export_data(self, id): ...
    async def send_notification(self, id, message): ...
    async def calculate_loyalty_points(self, id): ...
    # 500 more lines...
```

### Fix

Split into focused services by responsibility:

```python
# GOOD: Focused services
class UserService:          # CRUD operations
    async def create(self, data): ...
    async def update(self, id, data): ...
    async def delete(self, id): ...
    async def get_or_404(self, id): ...

class AuthService:          # Authentication
    async def login(self, credentials): ...
    async def logout(self, token): ...
    async def refresh_token(self, token): ...
    async def reset_password(self, email): ...

class ProfileService:       # Profile management
    async def update_avatar(self, user_id, file): ...
    async def export_data(self, user_id): ...

class LoyaltyService:       # Loyalty program
    async def calculate_points(self, user_id): ...
    async def redeem_points(self, user_id, points): ...
```

---

## Implicit Coupling via Global State

### Problem

Importing and mutating module-level variables creates hidden dependencies between components.

```python
# BAD: Global mutable state
# app/state.py
connected_clients: dict[str, WebSocket] = {}
request_counter: int = 0

# app/routers/ws.py
from app.state import connected_clients
connected_clients[client_id] = websocket  # Mutates global

# app/middleware.py
from app.state import request_counter
request_counter += 1  # Race condition in async code
```

### Fix

Use `app.state` for application-wide state, and inject via dependencies:

```python
# GOOD: State on app.state, injected via dependencies
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.connection_manager = ConnectionManager()
    yield

def get_connection_manager(request: Request) -> ConnectionManager:
    return request.app.state.connection_manager

@router.websocket("/ws/{client_id}")
async def ws_endpoint(
    websocket: WebSocket,
    client_id: str,
    manager: ConnectionManager = Depends(get_connection_manager),
):
    await manager.connect(client_id, websocket)
```

---

## Schema Inheritance Misuse

### Problem

Creating deeply nested inheritance hierarchies that make schemas hard to understand and maintain.

```python
# BAD: Deep inheritance chain
class BaseItem(BaseModel): ...
class ItemWithPrice(BaseItem): ...
class ItemWithCategory(ItemWithPrice): ...
class ItemWithStock(ItemWithCategory): ...
class ItemWithReviews(ItemWithStock): ...
class ProductResponse(ItemWithReviews): ...  # What fields does this have?
```

### Fix

Use flat composition. Limit inheritance to one level of base-create-update-response:

```python
# GOOD: Flat, predictable schema structure
class ProductBase(BaseModel):
    name: str
    price: Decimal
    category_id: int

class ProductCreate(ProductBase):
    sku: str

class ProductUpdate(BaseModel):  # Separate base, all optional
    name: str | None = None
    price: Decimal | None = None
    category_id: int | None = None

class ProductResponse(ProductBase):
    id: int
    sku: str
    is_active: bool
    created_at: datetime
    # Compose related data explicitly, not via inheritance
    category: CategoryResponse | None = None
    review_count: int = 0
```

---

## Missing Pagination on List Endpoints

### Problem

Returning unbounded lists from database queries leads to memory exhaustion and slow responses.

```python
# BAD: No pagination - returns entire table
@router.get("/products/")
async def list_products(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Product))
    return result.scalars().all()  # Could be millions of rows
```

### Fix

Always paginate list endpoints with reasonable defaults and maximums:

```python
# GOOD: Paginated with enforced limits
@router.get("/products/", response_model=PaginatedResponse[ProductResponse])
async def list_products(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),  # Max 100
    service: ProductService = Depends(),
):
    return await service.list_paginated(page=page, page_size=page_size)
```

---

## Async/Sync Session Mismatch

### Problem

Using a synchronous database session inside an `async def` endpoint or vice versa.

```python
# BAD: Sync session in async endpoint blocks the event loop
from sqlalchemy.orm import Session

async def get_db():
    with Session(sync_engine) as session:
        yield session  # Blocking I/O in async context

@router.get("/users/")
async def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()  # Blocks event loop
```

### Fix

Match the session type to the endpoint type:

```python
# GOOD: Async session for async endpoints
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        yield session

@router.get("/users/", response_model=list[UserResponse])
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()
```

---

## Testing Anti-Patterns

### Problem: Testing Implementation Instead of Behavior

```python
# BAD: Tests that mirror implementation details
async def test_create_user(client, db_session):
    response = await client.post("/users/", json=user_data)
    # Testing database state instead of API response
    result = await db_session.execute(select(User).where(User.email == "test@example.com"))
    user = result.scalar_one()
    assert user.hashed_password.startswith("$argon2")  # Knows too much about internals
```

### Fix: Test behavior through the public API

```python
# GOOD: Test behavior via API response
async def test_create_user(client):
    response = await client.post("/users/", json={
        "email": "test@example.com",
        "full_name": "Test User",
        "password": "Str0ngP@ss!",
    })
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert "password" not in data
    assert "hashed_password" not in data

    # Verify via GET endpoint
    get_response = await client.get(f"/users/{data['id']}")
    assert get_response.status_code == 200
```

### Problem: Not Cleaning Up Dependency Overrides

```python
# BAD: Override leaks between tests
def test_admin_access(client):
    app.dependency_overrides[get_current_user] = lambda: admin_user
    response = client.get("/admin/")
    assert response.status_code == 200
    # Missing: app.dependency_overrides.clear()

def test_unauthorized(client):
    # This test passes with admin_user from the previous test!
    response = client.get("/admin/")
    assert response.status_code == 403  # FAILS
```

### Fix: Use fixtures with automatic cleanup

```python
# GOOD: Fixture handles cleanup
@pytest.fixture(autouse=True)
def clean_overrides():
    yield
    app.dependency_overrides.clear()
```
