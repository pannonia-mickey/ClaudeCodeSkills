# Refactoring Guide for FastAPI Applications

This reference provides step-by-step refactoring recipes for improving FastAPI application structure. Each recipe targets a specific structural problem and includes incremental steps to apply safely.

---

## Recipe 1: Extract Service from Fat Endpoint

**When to apply**: Endpoint handler contains business logic, multiple database operations, or conditional branching beyond simple request/response mapping.

### Step 1: Identify the Business Logic

Look for code in the endpoint that is not about HTTP (parsing request, returning response). Everything else is a candidate for extraction.

```python
# Before: Fat endpoint
@router.post("/orders/", status_code=201)
async def create_order(
    order_in: OrderCreate,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
    background_tasks: BackgroundTasks,
):
    # --- Business logic starts ---
    for item in order_in.items:
        product = await db.get(Product, item.product_id)
        if not product:
            raise HTTPException(404, f"Product {item.product_id} not found")
        if product.stock < item.quantity:
            raise HTTPException(400, "Insufficient stock")

    order = OrderModel(user_id=user.id, status="pending", total=Decimal("0"))
    db.add(order)
    await db.flush()

    total = Decimal("0")
    for item in order_in.items:
        product = await db.get(Product, item.product_id)
        total += product.price * item.quantity
        product.stock -= item.quantity
        db.add(OrderItemModel(
            order_id=order.id,
            product_id=item.product_id,
            quantity=item.quantity,
            price=product.price,
        ))

    order.total = total
    await db.commit()
    # --- Business logic ends ---

    background_tasks.add_task(send_order_email, user.email, order.id)
    return order
```

### Step 2: Create the Service Class

Move business logic into a service class. The service receives its dependencies via `Depends()`.

```python
# app/services/order_service.py
from fastapi import Depends
from app.repositories.order_repo import OrderRepository
from app.repositories.product_repo import ProductRepository
from app.dependencies import get_current_user
from app.exceptions import NotFoundError, InsufficientStockError

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository = Depends(),
        product_repo: ProductRepository = Depends(),
        current_user: User = Depends(get_current_user),
    ):
        self.order_repo = order_repo
        self.product_repo = product_repo
        self.current_user = current_user

    async def place_order(self, data: OrderCreate) -> Order:
        # Validate stock
        for item in data.items:
            product = await self.product_repo.get_by_id(item.product_id)
            if not product:
                raise NotFoundError("Product", item.product_id)
            if product.stock < item.quantity:
                raise InsufficientStockError(product.name)

        # Create order with items
        total = Decimal("0")
        for item in data.items:
            product = await self.product_repo.get_by_id(item.product_id)
            total += product.price * item.quantity

        order = await self.order_repo.create(
            user_id=self.current_user.id,
            total=total,
            status="pending",
        )

        for item in data.items:
            product = await self.product_repo.get_by_id(item.product_id)
            await self.order_repo.add_item(
                order_id=order.id,
                product_id=item.product_id,
                quantity=item.quantity,
                price=product.price,
            )
            await self.product_repo.decrement_stock(item.product_id, item.quantity)

        return order
```

### Step 3: Slim Down the Endpoint

```python
# After: Thin endpoint
@router.post("/orders/", response_model=OrderResponse, status_code=201)
async def create_order(
    order_in: OrderCreate,
    service: OrderService = Depends(),
    background_tasks: BackgroundTasks,
):
    order = await service.place_order(order_in)
    background_tasks.add_task(send_order_email, order.user_id, order.id)
    return order
```

### Step 4: Write Tests for the Service

```python
# tests/services/test_order_service.py
@pytest.mark.anyio
async def test_place_order_validates_stock(order_service, product_factory):
    product = await product_factory(stock=2)
    data = OrderCreate(items=[{"product_id": product.id, "quantity": 5}])

    with pytest.raises(InsufficientStockError):
        await order_service.place_order(data)
```

---

## Recipe 2: Extract Repository from Service

**When to apply**: Service methods contain SQLAlchemy queries mixed with business logic.

### Step 1: Identify Data Access Code

```python
# Before: Queries embedded in service
class UserService:
    def __init__(self, db: AsyncSession = Depends(get_db)):
        self.db = db

    async def create(self, data: UserCreate) -> User:
        # Query mixed with business logic
        result = await self.db.execute(
            select(User).where(User.email == data.email)
        )
        if result.scalar_one_or_none():
            raise ConflictError("User", "email", data.email)

        user = User(
            email=data.email,
            full_name=data.full_name,
            hashed_password=hash_password(data.password),
        )
        self.db.add(user)
        await self.db.flush()
        await self.db.refresh(user)
        return user
```

### Step 2: Create Repository

```python
# app/repositories/user_repo.py
class UserRepository:
    def __init__(self, session: AsyncSession = Depends(get_db)):
        self.session = session

    async def get_by_id(self, id: int) -> User | None:
        return await self.session.get(User, id)

    async def get_by_email(self, email: str) -> User | None:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(self, **kwargs) -> User:
        user = User(**kwargs)
        self.session.add(user)
        await self.session.flush()
        await self.session.refresh(user)
        return user

    async def update_by_id(self, id: int, **kwargs) -> User | None:
        user = await self.get_by_id(id)
        if not user:
            return None
        for key, value in kwargs.items():
            if value is not None:
                setattr(user, key, value)
        await self.session.flush()
        await self.session.refresh(user)
        return user
```

### Step 3: Refactor Service to Use Repository

```python
# After: Service uses repository, no SQLAlchemy imports
class UserService:
    def __init__(self, repo: UserRepository = Depends()):
        self.repo = repo

    async def create(self, data: UserCreate) -> User:
        existing = await self.repo.get_by_email(data.email)
        if existing:
            raise ConflictError("User", "email", data.email)

        return await self.repo.create(
            email=data.email,
            full_name=data.full_name,
            hashed_password=hash_password(data.password),
        )
```

---

## Recipe 3: Replace HTTPException with Domain Exceptions

**When to apply**: `HTTPException` is raised outside of endpoint handlers (in services, repositories, or utilities).

### Step 1: Define Domain Exception Classes

```python
# app/exceptions.py
class AppError(Exception):
    """Base for all application errors."""
    pass

class NotFoundError(AppError):
    def __init__(self, resource: str, identifier: int | str):
        self.resource = resource
        self.identifier = identifier
        self.message = f"{resource} '{identifier}' not found"
        super().__init__(self.message)

class ConflictError(AppError):
    def __init__(self, resource: str, field: str, value: str):
        self.message = f"{resource} with {field} '{value}' already exists"
        super().__init__(self.message)

class ForbiddenError(AppError):
    def __init__(self, detail: str = "You do not have permission to perform this action"):
        self.message = detail
        super().__init__(self.message)

class ValidationError(AppError):
    def __init__(self, detail: str):
        self.message = detail
        super().__init__(self.message)
```

### Step 2: Register Exception Handlers

```python
# app/main.py
from app.exceptions import AppError, NotFoundError, ConflictError, ForbiddenError

@app.exception_handler(NotFoundError)
async def not_found_handler(request: Request, exc: NotFoundError):
    return JSONResponse(
        status_code=404,
        content={"error": "NOT_FOUND", "detail": exc.message},
    )

@app.exception_handler(ConflictError)
async def conflict_handler(request: Request, exc: ConflictError):
    return JSONResponse(
        status_code=409,
        content={"error": "CONFLICT", "detail": exc.message},
    )

@app.exception_handler(ForbiddenError)
async def forbidden_handler(request: Request, exc: ForbiddenError):
    return JSONResponse(
        status_code=403,
        content={"error": "FORBIDDEN", "detail": exc.message},
    )

@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=400,
        content={"error": "APP_ERROR", "detail": str(exc)},
    )
```

### Step 3: Replace HTTPException in Services

```python
# Before
async def get_order(self, order_id: int) -> Order:
    order = await self.repo.get_by_id(order_id)
    if not order:
        raise HTTPException(404, "Order not found")
    return order

# After
async def get_order(self, order_id: int) -> Order:
    order = await self.repo.get_by_id(order_id)
    if not order:
        raise NotFoundError("Order", order_id)
    return order
```

---

## Recipe 4: Introduce Annotated Types for Dependency Injection

**When to apply**: The same `Depends(...)` patterns are repeated across many endpoints, creating visual noise and duplication.

### Step 1: Identify Repeated Dependencies

```python
# Before: Same Depends() pattern repeated everywhere
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
): ...

@router.get("/users/")
async def list_users(
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
): ...

@router.patch("/users/{user_id}")
async def update_user(
    user_id: int,
    data: UserUpdate,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
): ...
```

### Step 2: Define Annotated Types

```python
# app/dependencies.py
from typing import Annotated

DbSession = Annotated[AsyncSession, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
AdminUser = Annotated[User, Depends(get_admin_user)]

# Common path/query parameter types
UserId = Annotated[int, Path(gt=0, description="User ID")]
Page = Annotated[int, Query(ge=1, description="Page number")]
PageSize = Annotated[int, Query(ge=1, le=100, description="Items per page")]
```

### Step 3: Refactor Endpoints

```python
# After: Clean, concise signatures
@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: UserId, db: DbSession, user: CurrentUser):
    ...

@router.get("/users/", response_model=PaginatedResponse[UserResponse])
async def list_users(
    page: Page = 1,
    page_size: PageSize = 20,
    db: DbSession,
    user: CurrentUser,
):
    ...
```

---

## Recipe 5: Convert Sync to Async Database Access

**When to apply**: Application was built with synchronous SQLAlchemy and needs to be migrated to async for better concurrency.

### Step 1: Update Dependencies

```bash
pip install sqlalchemy[asyncio] asyncpg  # or aiosqlite for SQLite
```

### Step 2: Migrate Engine and Session Factory

```python
# Before (sync)
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

engine = create_engine("postgresql://user:pass@localhost/db")
SessionLocal = sessionmaker(engine)

def get_db() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session

# After (async)
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
async_session_factory = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Step 3: Migrate Repository Queries

```python
# Before (sync)
def get_by_id(self, id: int) -> User | None:
    return self.session.query(User).filter(User.id == id).first()

def get_many(self, skip: int, limit: int) -> list[User]:
    return self.session.query(User).offset(skip).limit(limit).all()

# After (async) - use select() instead of query()
async def get_by_id(self, id: int) -> User | None:
    result = await self.session.execute(select(User).where(User.id == id))
    return result.scalar_one_or_none()

async def get_many(self, skip: int, limit: int) -> list[User]:
    result = await self.session.execute(select(User).offset(skip).limit(limit))
    return list(result.scalars().all())
```

### Step 4: Migrate Endpoints

```python
# Before (sync)
@router.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    ...

# After (async)
@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, service: UserService = Depends()):
    return await service.get_or_404(user_id)
```

---

## Recipe 6: Introduce Response Schemas for Existing Endpoints

**When to apply**: Endpoints return raw dicts or ORM objects without response models.

### Step 1: Audit Current Responses

Find endpoints missing `response_model`:

```python
# Find these patterns in the codebase:
@router.get("/users/{user_id}")           # No response_model
async def get_user(...):
    return user                            # Returns ORM object directly

@router.get("/products/")                 # No response_model
async def list_products(...):
    return {"items": products, "total": count}  # Returns raw dict
```

### Step 2: Create Response Schemas

```python
# app/schemas/users.py
from pydantic import BaseModel, ConfigDict
from datetime import datetime

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str
    full_name: str
    is_active: bool
    created_at: datetime
```

### Step 3: Add response_model to Endpoints

```python
# After: Explicit response model
@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, service: UserService = Depends()):
    return await service.get_or_404(user_id)
```

Verify that the response does not expose sensitive fields by running the endpoint and checking the JSON output against the schema.

---

## Recipe 7: Add Structured Logging

**When to apply**: Application uses `print()` or basic `logging` without structured context.

### Step 1: Install structlog

```bash
pip install structlog
```

### Step 2: Configure in Lifespan

```python
# app/logging.py
import structlog
import logging

def setup_logging(json_output: bool = True, log_level: str = "INFO"):
    processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
    ]
    if json_output:
        processors.append(structlog.processors.JSONRenderer())
    else:
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, log_level.upper())
        ),
    )
```

### Step 3: Replace print/logging Calls

```python
# Before
print(f"Creating order for user {user_id}")
logging.info(f"Order {order_id} created with total {total}")

# After
import structlog
logger = structlog.get_logger()

logger.info("creating_order", user_id=user_id)
logger.info("order_created", order_id=order_id, total=str(total))
```

### Step 4: Add Request Context Middleware

```python
# app/middleware.py
class RequestContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid4()))
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
        )
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

Now every log entry automatically includes `request_id`, `method`, and `path`.

---

## Code Review Checklist for FastAPI

When reviewing a FastAPI pull request, verify these items:

**Architecture**
- [ ] Endpoint handlers are thin (no business logic beyond parsing and returning)
- [ ] Business logic lives in service classes
- [ ] Database queries live in repository classes
- [ ] No circular imports between modules
- [ ] Layer dependencies flow downward only (router -> service -> repository)

**Type Safety**
- [ ] All function parameters and return types are annotated
- [ ] `response_model` is declared on every endpoint
- [ ] Pydantic v2 patterns used (`ConfigDict`, `field_validator`, not v1 `Config` class)
- [ ] No `Any` types unless absolutely necessary

**Error Handling**
- [ ] No bare `HTTPException` in services or repositories
- [ ] Custom exception classes used for domain errors
- [ ] Exception handlers registered for all custom exceptions
- [ ] No bare `except Exception` catching

**Security**
- [ ] Sensitive fields (password, tokens) excluded from response schemas
- [ ] IDOR prevention: ownership verified before returning resources
- [ ] Input validated via Pydantic schemas, not manual parsing
- [ ] File uploads validated for type and size

**Database**
- [ ] No N+1 queries (eager loading used for relationships)
- [ ] Session management via `Depends(get_db)`, not module-level
- [ ] Transactions scoped appropriately (service method or UoW)
- [ ] Pagination applied to all list endpoints

**Testing**
- [ ] Tests cover happy path and error paths
- [ ] Dependency overrides cleaned up after each test
- [ ] Tests verify API behavior (status codes, response shape), not implementation details
- [ ] No hardcoded IDs or timestamps that could cause flaky tests

**Async**
- [ ] No synchronous blocking calls inside `async def` functions
- [ ] `asyncio.gather` used for independent concurrent operations
- [ ] Async context managers used for resource lifecycle management
