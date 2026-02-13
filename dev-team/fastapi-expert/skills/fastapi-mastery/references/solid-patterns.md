# SOLID Principles in FastAPI

This reference provides concrete implementations of each SOLID principle within FastAPI applications, with production-ready Python code examples demonstrating how FastAPI's architecture naturally supports clean design.

---

## Single Responsibility Principle (SRP)

> A module should have one, and only one, reason to change.

### One Router Per Domain

Each router file handles endpoints for exactly one domain entity. Mixing user and product endpoints in a single router violates SRP because changes to user logic should never require touching product code.

```python
# app/routers/users.py - ONLY user-related endpoints
from fastapi import APIRouter, Depends
from app.schemas.users import UserCreate, UserResponse
from app.services.user_service import UserService

router = APIRouter(prefix="/users", tags=["Users"])

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(
    user_in: UserCreate,
    service: UserService = Depends(),
):
    return await service.create(user_in)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: UserService = Depends(),
):
    return await service.get_or_404(user_id)
```

### Single-Purpose Dependencies

Each dependency function does exactly one thing. Authentication, authorization, and database access are separate dependencies composed together rather than combined into one monolithic function.

```python
# BAD: One dependency doing too much
async def get_authenticated_admin_with_db(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> tuple[User, AsyncSession]:
    user = await authenticate(token, db)
    check_admin(user)
    return user, db

# GOOD: Separated concerns, composed via dependency chain
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    return await authenticate(token, db)

async def get_admin_user(
    user: User = Depends(get_current_user),
) -> User:
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="Admin access required")
    return user
```

### Service Layer Separation

Business logic lives in service classes, not in endpoint handlers. Endpoint handlers are thin: they parse input, call the service, and return the response.

```python
# app/services/user_service.py
class UserService:
    def __init__(self, repo: UserRepository = Depends()):
        self.repo = repo

    async def create(self, data: UserCreate) -> User:
        existing = await self.repo.get_by_email(data.email)
        if existing:
            raise ConflictError("User", "email", data.email)
        hashed = hash_password(data.password)
        return await self.repo.create(
            email=data.email,
            full_name=data.full_name,
            hashed_password=hashed,
        )
```

---

## Open/Closed Principle (OCP)

> Software entities should be open for extension but closed for modification.

### Middleware Extensibility

FastAPI's middleware stack allows adding cross-cutting behavior without modifying existing endpoint code. New middleware extends the request/response pipeline without touching handlers.

```python
# Adding observability without modifying any endpoint
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Extend behavior by adding middleware - no endpoint changes needed
app.add_middleware(CorrelationIdMiddleware)
app.add_middleware(RateLimitMiddleware, calls=100, period=60)
FastAPIInstrumentor.instrument_app(app)
```

### Dependency Overrides for Extension

FastAPI's `dependency_overrides` mechanism allows replacing behavior at runtime without modifying the original dependency. This is the framework's built-in OCP support.

```python
# Original dependency
async def get_email_sender() -> EmailSender:
    return SmtpEmailSender(settings.smtp_host)

# In testing or alternative deployments - extend without modifying
app.dependency_overrides[get_email_sender] = lambda: MockEmailSender()
```

### Plugin-Style Router Registration

New feature modules can be added by registering new routers without modifying existing ones:

```python
# app/main.py - open for extension via new routers
def register_routers(app: FastAPI) -> None:
    from app.routers import users, products, orders, analytics

    for router_module in [users, products, orders, analytics]:
        app.include_router(router_module.router, prefix="/api/v1")

# Adding a new domain only requires creating a new router module
# and adding it to the list - existing routers are never modified
```

---

## Liskov Substitution Principle (LSP)

> Subtypes must be substitutable for their base types.

### Pydantic Model Inheritance

Response models that inherit from a base must be usable anywhere the base is expected. Adding fields is safe; changing or removing field types breaks LSP.

```python
class ItemBase(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    name: str
    description: str
    price: Decimal

class ItemResponse(ItemBase):
    """Adds read-only fields. Substitutable for ItemBase."""
    id: int
    created_at: datetime
    category: CategoryResponse  # Additional field is safe

class ItemDetailResponse(ItemResponse):
    """Further enrichment. Substitutable for ItemResponse."""
    reviews: list[ReviewResponse]
    average_rating: float
    stock_quantity: int
```

### Consistent Response Models

All list endpoints return the same pagination wrapper structure. Consuming code can rely on the shape being consistent regardless of the resource type.

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
    has_next: bool

# Both endpoints return PaginatedResponse - consumers handle them identically
@router.get("/users", response_model=PaginatedResponse[UserResponse])
@router.get("/products", response_model=PaginatedResponse[ProductResponse])
```

---

## Interface Segregation Principle (ISP)

> Clients should not be forced to depend on interfaces they do not use.

### Targeted Dependencies

Instead of one large dependency that provides everything, create small, focused dependencies that endpoints request individually.

```python
# BAD: Fat dependency providing everything
class AppContext:
    def __init__(self):
        self.db = get_db()
        self.cache = get_cache()
        self.email = get_email()
        self.storage = get_storage()

# GOOD: Granular dependencies - endpoints take only what they need
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),  # Only needs DB
):
    ...

@router.post("/users/{user_id}/avatar")
async def upload_avatar(
    user_id: int,
    file: UploadFile,
    storage: StorageBackend = Depends(get_storage),  # Only needs storage
    db: AsyncSession = Depends(get_db),
):
    ...
```

### Granular Permission Dependencies

Rather than one permission check that validates all possible permissions, create focused permission dependencies for specific capabilities.

```python
# Fine-grained permission dependencies
require_read = PermissionChecker(["items:read"])
require_write = PermissionChecker(["items:write"])
require_delete = PermissionChecker(["items:delete"])
require_admin = PermissionChecker(["admin"])

@router.get("/items/", dependencies=[Depends(require_read)])
async def list_items(): ...

@router.post("/items/", dependencies=[Depends(require_write)])
async def create_item(): ...

@router.delete("/items/{id}", dependencies=[Depends(require_delete)])
async def delete_item(): ...
```

### Segregated Repository Interfaces

Define protocol classes for different data access patterns so services only depend on the operations they actually use:

```python
from typing import Protocol

class ReadRepository(Protocol[T]):
    async def get_by_id(self, id: int) -> T | None: ...
    async def get_many(self, skip: int, limit: int) -> list[T]: ...

class WriteRepository(Protocol[T]):
    async def create(self, **kwargs) -> T: ...
    async def update(self, id: int, **kwargs) -> T: ...
    async def delete(self, id: int) -> None: ...

class UserReadService:
    """Only depends on read operations."""
    def __init__(self, repo: ReadRepository[User]):
        self.repo = repo
```

---

## Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

### Depends() as Dependency Injection

FastAPI's `Depends()` is a first-class DI container. High-level services declare what they need via type hints and `Depends()`, and the framework resolves the concrete implementation.

```python
from typing import Protocol

class EmailSender(Protocol):
    async def send(self, to: str, subject: str, body: str) -> None: ...

class SmtpEmailSender:
    async def send(self, to: str, subject: str, body: str) -> None:
        # SMTP implementation
        ...

class SesEmailSender:
    async def send(self, to: str, subject: str, body: str) -> None:
        # AWS SES implementation
        ...

# The dependency function is the abstraction boundary
async def get_email_sender() -> EmailSender:
    if settings.email_backend == "ses":
        return SesEmailSender()
    return SmtpEmailSender()

# Service depends on abstraction, not concrete class
class NotificationService:
    def __init__(self, sender: EmailSender = Depends(get_email_sender)):
        self.sender = sender
```

### Abstract Repository Pattern

The repository abstraction decouples business logic from database technology. Services depend on the repository protocol, not on SQLAlchemy directly.

```python
from typing import Protocol, Generic, TypeVar
from abc import abstractmethod

T = TypeVar("T")

class BaseRepository(Protocol[T]):
    @abstractmethod
    async def get_by_id(self, id: int) -> T | None: ...

    @abstractmethod
    async def create(self, **kwargs) -> T: ...

    @abstractmethod
    async def update(self, id: int, **kwargs) -> T: ...

    @abstractmethod
    async def delete(self, id: int) -> bool: ...

class SqlAlchemyUserRepository:
    """Concrete implementation using SQLAlchemy."""
    def __init__(self, session: AsyncSession = Depends(get_db)):
        self.session = session

    async def get_by_id(self, id: int) -> User | None:
        result = await self.session.execute(
            select(UserModel).where(UserModel.id == id)
        )
        return result.scalar_one_or_none()

    async def create(self, **kwargs) -> User:
        user = UserModel(**kwargs)
        self.session.add(user)
        await self.session.flush()
        return user
```

### Protocol-Based Services

Using Python's `Protocol` class for structural subtyping enables true DIP without requiring explicit inheritance. Any class that implements the right methods satisfies the protocol.

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class CacheBackend(Protocol):
    async def get(self, key: str) -> str | None: ...
    async def set(self, key: str, value: str, ttl: int = 300) -> None: ...
    async def delete(self, key: str) -> None: ...

class RedisCacheBackend:
    """Satisfies CacheBackend protocol without inheriting from it."""
    def __init__(self, redis_client):
        self.redis = redis_client

    async def get(self, key: str) -> str | None:
        return await self.redis.get(key)

    async def set(self, key: str, value: str, ttl: int = 300) -> None:
        await self.redis.setex(key, ttl, value)

    async def delete(self, key: str) -> None:
        await self.redis.delete(key)

class InMemoryCacheBackend:
    """Alternative implementation for testing or simple deployments."""
    def __init__(self):
        self._store: dict[str, str] = {}

    async def get(self, key: str) -> str | None:
        return self._store.get(key)

    async def set(self, key: str, value: str, ttl: int = 300) -> None:
        self._store[key] = value

    async def delete(self, key: str) -> None:
        self._store.pop(key, None)

# Swap implementations without changing any service code
async def get_cache() -> CacheBackend:
    if settings.cache_backend == "redis":
        return RedisCacheBackend(redis_pool)
    return InMemoryCacheBackend()
```

---

## Applying SOLID Holistically

In a well-structured FastAPI application, SOLID principles reinforce each other:

- **SRP** keeps routers, services, and repositories focused on one domain.
- **OCP** allows adding middleware, routers, and dependencies without modifying existing code.
- **LSP** ensures Pydantic model hierarchies and response wrappers are consistently substitutable.
- **ISP** keeps dependencies small and targeted so endpoints only import what they use.
- **DIP** uses `Depends()` and `Protocol` to decouple high-level business logic from infrastructure.

Together, these principles produce FastAPI applications that are testable (dependencies can be overridden), maintainable (changes are localized), and extensible (new features plug in without disrupting existing behavior).
