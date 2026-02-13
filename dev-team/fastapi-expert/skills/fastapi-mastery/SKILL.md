---
name: FastAPI Mastery
description: This skill should be used when the user asks about "FastAPI endpoint", "FastAPI route", "FastAPI middleware", "FastAPI dependency", or "Pydantic model". It covers core FastAPI framework patterns including routing, dependency injection, middleware, exception handling, lifespan events, and Pydantic v2 integration. Use this skill for any task involving FastAPI endpoint creation, request/response modeling, application architecture decisions, or building and refactoring FastAPI applications.
---

### Application Bootstrap and Lifespan

To initialize a FastAPI application with managed resources, use the lifespan context manager pattern introduced in modern FastAPI:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: initialize resources
    app.state.db_pool = await create_async_pool()
    app.state.redis = await aioredis.from_url("redis://localhost")
    yield
    # Shutdown: release resources
    await app.state.db_pool.dispose()
    await app.state.redis.close()

app = FastAPI(
    title="My Service",
    version="1.0.0",
    lifespan=lifespan,
)
```

To avoid the deprecated `@app.on_event("startup")` and `@app.on_event("shutdown")` decorators, always prefer the lifespan approach. It provides a single, clear location for resource management and guarantees cleanup runs even on abnormal shutdown.

### Routing and Router Organization

To organize endpoints by domain, create separate `APIRouter` instances and include them in the main app:

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",
    tags=["Users"],
    responses={404: {"description": "User not found"}},
)

@router.get("/", summary="List users", response_model=list[UserResponse])
async def list_users(
    skip: int = Query(0, ge=0),
    limit: int = Query(20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    return await user_service.get_many(db, skip=skip, limit=limit)
```

To mount routers onto the application:

```python
from app.routers import users, products, orders

app.include_router(users.router)
app.include_router(products.router, prefix="/api/v1")
app.include_router(orders.router, prefix="/api/v1")
```

To keep routers focused, limit each router file to a single domain entity. Place shared query parameter models and path dependencies at the router level using the `dependencies` parameter.

### Dependency Injection with Depends()

To inject shared resources into endpoint handlers, define dependency functions and reference them with `Depends()`:

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        yield session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    payload = decode_jwt(token)
    user = await db.get(User, payload["sub"])
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return user
```

To create parameterized dependencies, use callable classes:

```python
class PermissionChecker:
    def __init__(self, required_permissions: list[str]):
        self.required_permissions = required_permissions

    async def __call__(
        self, user: User = Depends(get_current_user)
    ) -> User:
        if not all(p in user.permissions for p in self.required_permissions):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user

require_admin = PermissionChecker(["admin"])
```

To apply dependencies at the router or app level for cross-cutting concerns:

```python
app = FastAPI(dependencies=[Depends(verify_api_key)])
router = APIRouter(dependencies=[Depends(rate_limiter)])
```

### Pydantic v2 Models

To define request and response schemas, use Pydantic v2 with `ConfigDict`:

```python
from pydantic import BaseModel, ConfigDict, field_validator, Field
from datetime import datetime

class UserBase(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        str_strip_whitespace=True,
        str_min_length=1,
    )

    email: str = Field(..., json_schema_extra={"example": "user@example.com"})
    full_name: str = Field(..., min_length=2, max_length=100)

class UserCreate(UserBase):
    password: str = Field(..., min_length=8)

    @field_validator("password")
    @classmethod
    def validate_password_strength(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain an uppercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain a digit")
        return v

class UserResponse(UserBase):
    id: int
    created_at: datetime
    is_active: bool
```

To create partial update schemas where all fields are optional, use a reusable pattern:

```python
from typing import Optional

class UserUpdate(BaseModel):
    email: Optional[str] = None
    full_name: Optional[str] = None

    model_config = ConfigDict(str_strip_whitespace=True)
```

To validate relationships between fields, use `model_validator`:

```python
from pydantic import model_validator

class DateRangeFilter(BaseModel):
    start_date: date
    end_date: date

    @model_validator(mode="after")
    def validate_date_range(self) -> "DateRangeFilter":
        if self.end_date < self.start_date:
            raise ValueError("end_date must be after start_date")
        return self
```

### Middleware

To add middleware for cross-cutting concerns, use the `@app.middleware("http")` decorator or class-based ASGI middleware:

```python
import time
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start
        response.headers["X-Process-Time"] = f"{duration:.4f}"
        return response

app.add_middleware(TimingMiddleware)
```

To configure CORS properly:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Note that middleware execution order is bottom-to-top: the last middleware added with `add_middleware` runs first (outermost).

### Exception Handling

To handle errors consistently across the application, define custom exceptions and register handlers:

```python
class AppException(Exception):
    def __init__(self, status_code: int, detail: str, error_code: str):
        self.status_code = status_code
        self.detail = detail
        self.error_code = error_code

class NotFoundError(AppException):
    def __init__(self, resource: str, resource_id: str):
        super().__init__(
            status_code=404,
            detail=f"{resource} with id '{resource_id}' not found",
            error_code="RESOURCE_NOT_FOUND",
        )

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error_code,
            "detail": exc.detail,
            "path": str(request.url),
        },
    )
```

### Background Tasks

To run non-blocking work after returning a response, use `BackgroundTasks`:

```python
from fastapi import BackgroundTasks

async def send_welcome_email(email: str, name: str):
    await email_client.send(to=email, template="welcome", context={"name": name})

@router.post("/users/", status_code=201, response_model=UserResponse)
async def create_user(
    user_in: UserCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    user = await user_service.create(db, user_in)
    background_tasks.add_task(send_welcome_email, user.email, user.full_name)
    return user
```

To choose between `BackgroundTasks` and a task queue like Celery: use `BackgroundTasks` for lightweight, fire-and-forget work (sending emails, logging). Use Celery or ARQ for long-running, retryable, or distributed tasks.

### WebSocket Endpoints

To implement WebSocket communication with connection management:

```python
from fastapi import WebSocket, WebSocketDisconnect

class ConnectionManager:
    def __init__(self):
        self.active_connections: dict[str, WebSocket] = {}

    async def connect(self, websocket: WebSocket, client_id: str):
        await websocket.accept()
        self.active_connections[client_id] = websocket

    def disconnect(self, client_id: str):
        self.active_connections.pop(client_id, None)

    async def broadcast(self, message: dict):
        for ws in self.active_connections.values():
            await ws.send_json(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket, client_id)
    try:
        while True:
            data = await websocket.receive_json()
            await manager.broadcast({"sender": client_id, **data})
    except WebSocketDisconnect:
        manager.disconnect(client_id)
```

### Project Structure

To organize a production FastAPI application, follow this layout:

```
app/
    __init__.py
    main.py              # FastAPI app instance, lifespan, middleware
    config.py            # pydantic-settings configuration
    dependencies.py      # Shared dependencies (get_db, get_current_user)
    exceptions.py        # Custom exceptions and handlers
    routers/
        __init__.py
        users.py
        products.py
    schemas/
        __init__.py
        users.py         # Pydantic models for user domain
        products.py
    services/
        __init__.py
        user_service.py  # Business logic layer
    repositories/
        __init__.py
        user_repo.py     # Database access layer
    models/
        __init__.py
        user.py          # SQLAlchemy ORM models
```

To keep the architecture clean, enforce a strict dependency direction: routers depend on schemas and services, services depend on repositories, repositories depend on models. Never import routers from services or repositories.

## References

- [references/solid-patterns.md](references/solid-patterns.md) - SOLID principles applied to FastAPI with concrete Python examples
- [references/best-practices.md](references/best-practices.md) - Async patterns, configuration, logging, error handling, and OpenAPI customization
