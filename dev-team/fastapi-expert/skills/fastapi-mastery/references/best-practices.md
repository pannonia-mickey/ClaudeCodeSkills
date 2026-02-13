# FastAPI Best Practices

This reference covers production-grade patterns for async handling, project configuration, structured logging, error management, and OpenAPI schema customization in FastAPI applications.

---

## Async Patterns

### When to Use async def vs def

FastAPI handles `async def` and `def` differently. Understanding this distinction is critical for performance.

```python
# Use async def for I/O-bound operations (database, HTTP calls, file I/O)
@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()

# Use def for CPU-bound operations - FastAPI runs these in a threadpool
@router.post("/reports/generate")
def generate_report(params: ReportParams):
    # Heavy computation - runs in threadpool, won't block the event loop
    return compute_report(params)
```

Never call synchronous blocking functions inside `async def` without wrapping them:

```python
import asyncio
from functools import partial

async def run_sync_in_threadpool(func, *args, **kwargs):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, partial(func, *args, **kwargs))

@router.post("/process")
async def process_data(data: ProcessInput):
    # Offload blocking call to threadpool
    result = await run_sync_in_threadpool(heavy_sync_function, data.payload)
    return {"result": result}
```

### Async Context Managers for Resource Management

Use async context managers to ensure resources are properly cleaned up:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_http_client():
    async with httpx.AsyncClient(timeout=30.0) as client:
        yield client

async def get_client() -> AsyncGenerator[httpx.AsyncClient, None]:
    async with get_http_client() as client:
        yield client

@router.get("/external-data")
async def fetch_external(client: httpx.AsyncClient = Depends(get_client)):
    response = await client.get("https://api.example.com/data")
    response.raise_for_status()
    return response.json()
```

### Concurrency Patterns

Use `asyncio.gather` for concurrent independent I/O operations:

```python
@router.get("/dashboard/{user_id}")
async def get_dashboard(
    user_id: int,
    db: AsyncSession = Depends(get_db),
):
    # Run independent queries concurrently
    user_task = get_user(db, user_id)
    orders_task = get_recent_orders(db, user_id)
    notifications_task = get_unread_notifications(db, user_id)

    user, orders, notifications = await asyncio.gather(
        user_task, orders_task, notifications_task
    )
    return DashboardResponse(
        user=user, orders=orders, notifications=notifications
    )
```

Use semaphores to limit concurrent access to rate-limited resources:

```python
import asyncio

external_api_semaphore = asyncio.Semaphore(10)

async def call_external_api(payload: dict) -> dict:
    async with external_api_semaphore:
        async with httpx.AsyncClient() as client:
            response = await client.post("https://api.example.com", json=payload)
            return response.json()
```

---

## Project Configuration with pydantic-settings

Use `pydantic-settings` for type-safe, validated configuration loaded from environment variables and `.env` files:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, field_validator

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="APP_",
        case_sensitive=False,
    )

    # Application
    app_name: str = "MyService"
    debug: bool = False
    environment: str = "production"

    # Database
    database_url: str
    db_pool_size: int = Field(default=5, ge=1, le=50)
    db_max_overflow: int = Field(default=10, ge=0)

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Security
    secret_key: str
    access_token_expire_minutes: int = 30
    allowed_origins: list[str] = ["http://localhost:3000"]

    @field_validator("environment")
    @classmethod
    def validate_environment(cls, v: str) -> str:
        allowed = {"development", "staging", "production", "testing"}
        if v not in allowed:
            raise ValueError(f"environment must be one of {allowed}")
        return v

# Singleton pattern using lru_cache
from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

Inject settings as a dependency:

```python
@router.get("/health")
async def health_check(settings: Settings = Depends(get_settings)):
    return {
        "status": "healthy",
        "environment": settings.environment,
        "version": settings.app_name,
    }
```

---

## Structured Logging

Use `structlog` for JSON-formatted, context-rich logging that integrates with observability platforms:

```python
import structlog
from uuid import uuid4

def setup_logging(json_output: bool = True):
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
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    )

logger = structlog.get_logger()
```

Add request context via middleware:

```python
import structlog
from uuid import uuid4

class RequestContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid4()))
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
        )

        logger.info("request_started")
        response = await call_next(request)
        logger.info("request_completed", status_code=response.status_code)

        response.headers["X-Request-ID"] = request_id
        return response
```

Log within services with bound context:

```python
class OrderService:
    def __init__(self):
        self.logger = structlog.get_logger()

    async def create_order(self, user_id: int, items: list[OrderItem]) -> Order:
        self.logger.info("creating_order", user_id=user_id, item_count=len(items))
        order = await self.repo.create(user_id=user_id, items=items)
        self.logger.info("order_created", order_id=order.id, total=str(order.total))
        return order
```

---

## Error Handling

### Hierarchical Exception System

Build a structured exception hierarchy for consistent, informative error responses:

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class AppError(Exception):
    """Base application error."""
    def __init__(
        self,
        status_code: int = 500,
        error_code: str = "INTERNAL_ERROR",
        detail: str = "An unexpected error occurred",
        headers: dict[str, str] | None = None,
    ):
        self.status_code = status_code
        self.error_code = error_code
        self.detail = detail
        self.headers = headers

class NotFoundError(AppError):
    def __init__(self, resource: str, identifier: str | int):
        super().__init__(
            status_code=404,
            error_code="NOT_FOUND",
            detail=f"{resource} '{identifier}' not found",
        )

class ConflictError(AppError):
    def __init__(self, resource: str, field: str, value: str):
        super().__init__(
            status_code=409,
            error_code="CONFLICT",
            detail=f"{resource} with {field} '{value}' already exists",
        )

class ValidationError(AppError):
    def __init__(self, errors: list[dict]):
        super().__init__(
            status_code=422,
            error_code="VALIDATION_ERROR",
            detail="Request validation failed",
        )
        self.errors = errors
```

Register exception handlers:

```python
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    logger.warning(
        "app_error",
        error_code=exc.error_code,
        detail=exc.detail,
        path=str(request.url),
    )
    content = {"error": exc.error_code, "detail": exc.detail}
    if isinstance(exc, ValidationError):
        content["errors"] = exc.errors
    return JSONResponse(
        status_code=exc.status_code,
        content=content,
        headers=exc.headers,
    )

@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("unhandled_error", path=str(request.url))
    return JSONResponse(
        status_code=500,
        content={"error": "INTERNAL_ERROR", "detail": "An unexpected error occurred"},
    )
```

### Validation Error Customization

Override FastAPI's default validation error format for consistency:

```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(
    request: Request, exc: RequestValidationError
) -> JSONResponse:
    errors = []
    for error in exc.errors():
        errors.append({
            "field": ".".join(str(loc) for loc in error["loc"]),
            "message": error["msg"],
            "type": error["type"],
        })
    return JSONResponse(
        status_code=422,
        content={
            "error": "VALIDATION_ERROR",
            "detail": "Request validation failed",
            "errors": errors,
        },
    )
```

---

## OpenAPI Customization

### Schema Metadata

Enrich the generated OpenAPI schema with descriptions, examples, and tags:

```python
app = FastAPI(
    title="E-Commerce API",
    description="API for managing products, orders, and users.",
    version="2.1.0",
    contact={"name": "API Team", "email": "api-team@example.com"},
    license_info={"name": "MIT"},
    openapi_tags=[
        {"name": "Users", "description": "User management operations"},
        {"name": "Products", "description": "Product catalog operations"},
        {"name": "Orders", "description": "Order processing and tracking"},
    ],
)
```

### Response Documentation

Document all possible responses on endpoints:

```python
@router.post(
    "/",
    response_model=OrderResponse,
    status_code=201,
    summary="Create a new order",
    description="Creates an order from the items in the user's cart.",
    responses={
        201: {"description": "Order created successfully"},
        400: {
            "description": "Invalid order (e.g., empty cart)",
            "content": {
                "application/json": {
                    "example": {"error": "BAD_REQUEST", "detail": "Cart is empty"}
                }
            },
        },
        409: {"description": "Insufficient stock for one or more items"},
    },
)
async def create_order(
    order_in: OrderCreate,
    user: User = Depends(get_current_user),
    service: OrderService = Depends(),
) -> OrderResponse:
    return await service.create(user.id, order_in)
```

### Schema Examples in Pydantic Models

Provide realistic examples directly on models using `json_schema_extra`:

```python
class ProductCreate(BaseModel):
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {
                    "name": "Wireless Mouse",
                    "description": "Ergonomic wireless mouse with USB-C receiver",
                    "price": 29.99,
                    "category_id": 5,
                    "sku": "WM-2024-001",
                }
            ]
        }
    )

    name: str = Field(..., min_length=1, max_length=200)
    description: str = Field("", max_length=2000)
    price: Decimal = Field(..., gt=0, decimal_places=2)
    category_id: int = Field(..., gt=0)
    sku: str = Field(..., pattern=r"^[A-Z]{2}-\d{4}-\d{3}$")
```

### Custom OpenAPI Schema Modification

Modify the generated schema programmatically for advanced use cases:

```python
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )

    # Add security scheme
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
        }
    }
    # Apply globally
    openapi_schema["security"] = [{"BearerAuth": []}]

    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

---

## Health Checks and Observability

Implement comprehensive health checks that verify dependencies:

```python
@router.get("/health", tags=["System"])
async def health_check(
    db: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
):
    checks = {}

    try:
        await db.execute(text("SELECT 1"))
        checks["database"] = "healthy"
    except Exception:
        checks["database"] = "unhealthy"

    try:
        await redis.ping()
        checks["redis"] = "healthy"
    except Exception:
        checks["redis"] = "unhealthy"

    all_healthy = all(v == "healthy" for v in checks.values())
    return JSONResponse(
        status_code=200 if all_healthy else 503,
        content={"status": "healthy" if all_healthy else "degraded", "checks": checks},
    )
```

---

## Performance Considerations

### Response Caching

Use caching headers and in-memory caches for read-heavy endpoints:

```python
from fastapi.responses import Response

@router.get("/products/{product_id}")
async def get_product(
    product_id: int,
    response: Response,
    cache: CacheBackend = Depends(get_cache),
    db: AsyncSession = Depends(get_db),
):
    cached = await cache.get(f"product:{product_id}")
    if cached:
        response.headers["X-Cache"] = "HIT"
        return json.loads(cached)

    product = await product_repo.get_by_id(db, product_id)
    if not product:
        raise NotFoundError("Product", product_id)

    await cache.set(f"product:{product_id}", product.model_dump_json(), ttl=300)
    response.headers["X-Cache"] = "MISS"
    response.headers["Cache-Control"] = "public, max-age=300"
    return product
```

### Streaming Responses

Use streaming for large payloads to avoid memory pressure:

```python
from fastapi.responses import StreamingResponse
import csv
import io

@router.get("/exports/users")
async def export_users(db: AsyncSession = Depends(get_db)):
    async def generate():
        output = io.StringIO()
        writer = csv.writer(output)
        writer.writerow(["id", "email", "full_name", "created_at"])
        yield output.getvalue()
        output.seek(0)
        output.truncate(0)

        async for batch in user_repo.stream_all(db, batch_size=1000):
            for user in batch:
                writer.writerow([user.id, user.email, user.full_name, user.created_at])
            yield output.getvalue()
            output.seek(0)
            output.truncate(0)

    return StreamingResponse(
        generate(),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=users.csv"},
    )
```
