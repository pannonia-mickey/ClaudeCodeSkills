---
name: FastAPI Clean Code
description: This skill should be used when the user asks about "FastAPI anti-pattern", "FastAPI code smell", "FastAPI refactor", "clean code FastAPI", "FastAPI best practice", "FastAPI naming convention", "type annotation FastAPI", "FastAPI code review", or "FastAPI code quality". It covers common anti-patterns with before/after refactoring examples, naming conventions, code smell detection, type annotation best practices, dependency injection patterns, service extraction, error handling design, and code review guidelines specific to FastAPI applications. Use this skill for any task involving code quality improvement, refactoring existing FastAPI code, reviewing pull requests, or establishing coding standards for a FastAPI project.
---

### Anti-Pattern: Fat Endpoints

The most common FastAPI anti-pattern is putting business logic, database queries, and side effects directly in endpoint handlers. Fat endpoints are hard to test, reuse, and maintain.

```python
# BAD: Fat endpoint with business logic, DB queries, and side effects
@router.post("/orders/", status_code=201)
async def create_order(
    order_in: OrderCreate,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
    background_tasks: BackgroundTasks,
):
    # Business validation mixed into handler
    for item in order_in.items:
        product = await db.execute(
            select(Product).where(Product.id == item.product_id)
        )
        product = product.scalar_one_or_none()
        if not product:
            raise HTTPException(404, f"Product {item.product_id} not found")
        if product.stock < item.quantity:
            raise HTTPException(400, f"Insufficient stock for {product.name}")

    # Database writes mixed into handler
    total = Decimal("0")
    order = OrderModel(user_id=user.id, status="pending")
    db.add(order)
    await db.flush()

    for item in order_in.items:
        product = (await db.execute(
            select(Product).where(Product.id == item.product_id)
        )).scalar_one()
        total += product.price * item.quantity
        db.add(OrderItemModel(
            order_id=order.id, product_id=item.product_id,
            quantity=item.quantity, price=product.price,
        ))
        product.stock -= item.quantity

    order.total = total
    await db.commit()

    # Side effects mixed into handler
    background_tasks.add_task(send_order_email, user.email, order.id)
    return order
```

```python
# GOOD: Thin endpoint delegates to service
@router.post("/orders/", response_model=OrderResponse, status_code=201)
async def create_order(
    order_in: OrderCreate,
    service: OrderService = Depends(),
):
    return await service.place_order(order_in)
```

The service encapsulates all business logic, making it independently testable and reusable across endpoints, CLI commands, or background jobs.

### Anti-Pattern: Circular Dependencies

When services import each other directly, circular imports occur. Break cycles using dependency injection and interfaces.

```python
# BAD: Circular import between services
# user_service.py
from app.services.order_service import OrderService  # Circular!

class UserService:
    async def delete_user(self, user_id: int):
        order_service = OrderService()
        await order_service.cancel_user_orders(user_id)
        await self.repo.delete(user_id)

# order_service.py
from app.services.user_service import UserService  # Circular!

class OrderService:
    async def place_order(self, data):
        user_service = UserService()
        user = await user_service.get_by_id(data.user_id)
```

```python
# GOOD: Break cycle with dependency injection
class UserService:
    def __init__(
        self,
        user_repo: UserRepository = Depends(),
        order_repo: OrderRepository = Depends(),
    ):
        self.user_repo = user_repo
        self.order_repo = order_repo

    async def delete_user(self, user_id: int):
        await self.order_repo.cancel_by_user(user_id)
        await self.user_repo.delete(user_id)
```

Alternatively, use domain events to decouple the interaction entirely.

### Anti-Pattern: Leaking ORM Models to API

Returning SQLAlchemy models directly from endpoints exposes internal database structure, allows accidental data leaks, and couples the API contract to the database schema.

```python
# BAD: ORM model as response
@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    return user  # Exposes hashed_password, internal IDs, etc.
```

```python
# GOOD: Explicit response schema
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str
    full_name: str
    is_active: bool
    created_at: datetime

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, service: UserService = Depends()):
    return await service.get_or_404(user_id)
```

### Anti-Pattern: Bare HTTPException Everywhere

Scattering `HTTPException` calls throughout services and repositories couples business logic to HTTP concepts.

```python
# BAD: HTTP concepts in the service layer
class OrderService:
    async def get_order(self, order_id: int) -> Order:
        order = await self.repo.get_by_id(order_id)
        if not order:
            raise HTTPException(status_code=404, detail="Order not found")  # HTTP in service!
        if order.user_id != self.current_user.id:
            raise HTTPException(status_code=403, detail="Not your order")  # HTTP in service!
        return order
```

```python
# GOOD: Domain exceptions mapped to HTTP at the boundary
class OrderService:
    async def get_order(self, order_id: int) -> Order:
        order = await self.repo.get_by_id(order_id)
        if not order:
            raise NotFoundError("Order", order_id)
        if order.user_id != self.current_user.id:
            raise NotFoundError("Order", order_id)  # Return 404, not 403, to hide existence
        return order

# Exception handler in main.py maps NotFoundError -> 404
@app.exception_handler(NotFoundError)
async def not_found_handler(request, exc):
    return JSONResponse(status_code=404, content={"error": "NOT_FOUND", "detail": exc.message})
```

### Naming Conventions

**Routers**: Use plural nouns for the file name and URL prefix. The file represents a collection of endpoints for one resource.

```python
# File: app/routers/products.py
router = APIRouter(prefix="/products", tags=["Products"])
```

**Endpoint functions**: Use verb-noun format describing the action.

```python
# GOOD: Clear, action-oriented names
async def list_products(...)       # GET /products/
async def create_product(...)      # POST /products/
async def get_product(...)         # GET /products/{id}
async def update_product(...)      # PATCH /products/{id}
async def delete_product(...)      # DELETE /products/{id}
async def search_products(...)     # GET /products/search

# BAD: Vague or redundant names
async def products(...)            # Which operation?
async def product_endpoint(...)    # Redundant suffix
async def handle_product(...)      # Ambiguous
async def do_create(...)           # Missing the noun
```

**Schemas**: Use a consistent suffix pattern to distinguish purpose.

```python
# Input schemas
class ProductCreate(BaseModel): ...    # For POST body
class ProductUpdate(BaseModel): ...    # For PATCH body
class ProductFilters(BaseModel): ...   # For query parameters

# Output schemas
class ProductResponse(BaseModel): ...        # Standard response
class ProductDetailResponse(BaseModel): ...  # Extended detail view
class ProductListItem(BaseModel): ...        # Compact list item

# Internal DTOs
class ProductDTO(BaseModel): ...       # Inter-service transfer
```

**Services**: Use the pattern `{Entity}Service` with methods named for the business operation.

```python
class OrderService:
    async def place_order(self, data: OrderCreate) -> Order: ...
    async def cancel_order(self, order_id: int, reason: str) -> None: ...
    async def get_or_404(self, order_id: int) -> Order: ...

# BAD: Generic or implementation-leaking names
class OrderManager: ...          # "Manager" is vague
class OrderHandler: ...          # Confusable with FastAPI handler
class OrderController: ...       # Not a Python/FastAPI convention
```

**Repositories**: Use the pattern `{Entity}Repository` with data-access-oriented methods.

```python
class ProductRepository:
    async def get_by_id(self, id: int) -> Product | None: ...
    async def get_by_sku(self, sku: str) -> Product | None: ...
    async def get_many(self, skip: int, limit: int) -> list[Product]: ...
    async def create(self, **kwargs) -> Product: ...
    async def update_by_id(self, id: int, **kwargs) -> Product: ...
    async def delete_by_id(self, id: int) -> bool: ...
    async def exists(self, **kwargs) -> bool: ...
```

**Dependencies**: Use the `get_` prefix for dependency provider functions.

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]: ...
async def get_current_user() -> User: ...
async def get_settings() -> Settings: ...
async def get_redis() -> Redis: ...
```

### Type Annotation Best Practices

Annotate every function signature, dependency, and return type. FastAPI relies on type annotations for request parsing, dependency injection, and OpenAPI schema generation.

```python
# GOOD: Fully annotated
from typing import Annotated

UserId = Annotated[int, Path(gt=0, description="User ID")]
PageSize = Annotated[int, Query(ge=1, le=100, description="Items per page")]

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: UserId,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    ...

# GOOD: Reusable annotated types reduce repetition
CurrentUser = Annotated[User, Depends(get_current_user)]
DbSession = Annotated[AsyncSession, Depends(get_db)]

@router.get("/me", response_model=UserResponse)
async def get_me(user: CurrentUser) -> UserResponse:
    return UserResponse.model_validate(user)
```

```python
# BAD: Missing annotations
@router.get("/users/{user_id}")
async def get_user(user_id, db=Depends(get_db)):  # No type hints
    ...

# BAD: Using Any or overly broad types
from typing import Any

async def process(data: Any) -> Any:  # Defeats type checking
    ...
```

### Dependency Injection Patterns

**Callable classes** for parameterized dependencies:

```python
# GOOD: Parameterized dependency using callable class
class RateLimiter:
    def __init__(self, calls: int, period: int):
        self.calls = calls
        self.period = period

    async def __call__(self, request: Request) -> None:
        key = f"rate:{request.client.host}"
        # Rate limit logic...

# Create specific instances
api_rate_limit = RateLimiter(calls=100, period=60)
auth_rate_limit = RateLimiter(calls=5, period=60)

router = APIRouter(dependencies=[Depends(api_rate_limit)])

@router.post("/auth/token", dependencies=[Depends(auth_rate_limit)])
async def login(): ...
```

**Yield dependencies** for resource cleanup:

```python
# GOOD: Yield dependency for resource lifecycle
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# GOOD: Yield dependency for HTTP client
async def get_http_client() -> AsyncGenerator[httpx.AsyncClient, None]:
    async with httpx.AsyncClient(timeout=30.0) as client:
        yield client
```

**Service injection** via `Depends()` on class `__init__`:

```python
# GOOD: Service with injected dependencies
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
```

### Error Handling Design

Build a three-tier error hierarchy:

1. **Domain errors**: Business rule violations (no HTTP concepts)
2. **Application errors**: Application-level errors with error codes
3. **HTTP mapping**: Centralized exception handlers translate to HTTP responses

```python
# Tier 1: Domain errors
class DomainError(Exception):
    def __init__(self, message: str):
        self.message = message

class NotFoundError(DomainError):
    def __init__(self, resource: str, identifier: int | str):
        super().__init__(f"{resource} '{identifier}' not found")
        self.resource = resource
        self.identifier = identifier

class ConflictError(DomainError):
    def __init__(self, resource: str, field: str, value: str):
        super().__init__(f"{resource} with {field} '{value}' already exists")

# Tier 2: Application errors
class AppError(Exception):
    def __init__(self, status_code: int, error_code: str, detail: str):
        self.status_code = status_code
        self.error_code = error_code
        self.detail = detail

# Tier 3: HTTP mapping (in main.py)
@app.exception_handler(NotFoundError)
async def handle_not_found(request: Request, exc: NotFoundError):
    return JSONResponse(status_code=404, content={
        "error": "NOT_FOUND", "detail": exc.message
    })

@app.exception_handler(ConflictError)
async def handle_conflict(request: Request, exc: ConflictError):
    return JSONResponse(status_code=409, content={
        "error": "CONFLICT", "detail": exc.message
    })

@app.exception_handler(DomainError)
async def handle_domain_error(request: Request, exc: DomainError):
    return JSONResponse(status_code=400, content={
        "error": "DOMAIN_ERROR", "detail": exc.message
    })
```

### Schema Design Principles

**Separate input from output schemas**. Never reuse a single model for both request body and response.

```python
# GOOD: Distinct schemas for distinct purposes
class UserCreate(BaseModel):
    email: str
    full_name: str
    password: str  # Input only, never in response

class UserUpdate(BaseModel):
    email: str | None = None
    full_name: str | None = None

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: str
    full_name: str
    is_active: bool
    created_at: datetime
    # No password field
```

**Use `model_dump(exclude_unset=True)` for partial updates**:

```python
async def update(self, user_id: int, data: UserUpdate) -> User:
    updates = data.model_dump(exclude_unset=True)  # Only fields the client sent
    if not updates:
        raise DomainError("No fields to update")
    return await self.repo.update_by_id(user_id, **updates)
```

**Define shared base schemas** to reduce duplication:

```python
class ProductBase(BaseModel):
    model_config = ConfigDict(from_attributes=True, str_strip_whitespace=True)
    name: str = Field(..., min_length=1, max_length=200)
    description: str = Field("", max_length=5000)
    price: Decimal = Field(..., gt=0, decimal_places=2)

class ProductCreate(ProductBase):
    sku: str = Field(..., pattern=r"^[A-Z]{2,4}-\d{4}-\d{3}$")

class ProductResponse(ProductBase):
    id: int
    sku: str
    is_active: bool
    created_at: datetime
```

## References

- [references/anti-patterns.md](references/anti-patterns.md) - Comprehensive catalog of FastAPI anti-patterns with before/after refactoring examples
- [references/refactoring-guide.md](references/refactoring-guide.md) - Step-by-step refactoring recipes for extracting services, decoupling layers, and improving testability
