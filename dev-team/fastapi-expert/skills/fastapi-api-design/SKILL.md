---
name: FastAPI API Design
description: This skill should be used when the user asks about "FastAPI API design", "FastAPI response model", "FastAPI pagination", or "OpenAPI FastAPI". It covers response model design, HTTP status code conventions, pagination strategies, filtering and sorting, API versioning, and OpenAPI documentation. Use this skill for any task involving API contract design, endpoint naming, response envelope patterns, or documentation generation in FastAPI.
---

### RESTful Endpoint Design

To design consistent, predictable API endpoints, follow REST conventions for resource naming and HTTP method semantics:

```python
from fastapi import APIRouter, Depends, Query, Path, status

router = APIRouter(prefix="/api/v1/products", tags=["Products"])

# Collection endpoints
@router.get("/", response_model=PaginatedResponse[ProductResponse], summary="List products")
async def list_products(
    page: int = Query(1, ge=1, description="Page number"),
    page_size: int = Query(20, ge=1, le=100, description="Items per page"),
    category_id: int | None = Query(None, description="Filter by category"),
    service: ProductService = Depends(),
):
    return await service.list(page=page, page_size=page_size, category_id=category_id)

@router.post("/", response_model=ProductResponse, status_code=status.HTTP_201_CREATED, summary="Create a product")
async def create_product(
    product_in: ProductCreate,
    service: ProductService = Depends(),
):
    return await service.create(product_in)

# Item endpoints
@router.get("/{product_id}", response_model=ProductDetailResponse, summary="Get product details")
async def get_product(
    product_id: int = Path(..., gt=0, description="Product ID"),
    service: ProductService = Depends(),
):
    return await service.get_or_404(product_id)

@router.patch("/{product_id}", response_model=ProductResponse, summary="Update a product")
async def update_product(
    product_id: int = Path(..., gt=0),
    product_in: ProductUpdate = ...,
    service: ProductService = Depends(),
):
    return await service.update(product_id, product_in)

@router.delete("/{product_id}", status_code=status.HTTP_204_NO_CONTENT, summary="Delete a product")
async def delete_product(
    product_id: int = Path(..., gt=0),
    service: ProductService = Depends(),
):
    await service.delete(product_id)
```

Naming conventions to follow:
- Use plural nouns for collections: `/products`, `/users`, `/orders`.
- Use path parameters for specific resources: `/products/{product_id}`.
- Use query parameters for filtering, sorting, and pagination.
- Use nested paths for sub-resources: `/users/{user_id}/orders`.
- Avoid verbs in URLs; let HTTP methods convey the action.

### Response Models

To design a consistent response contract, define clear input and output schemas:

```python
from pydantic import BaseModel, ConfigDict, Field
from datetime import datetime
from decimal import Decimal

class ProductBase(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    name: str = Field(..., min_length=1, max_length=200)
    description: str = Field("", max_length=5000)
    price: Decimal = Field(..., gt=0, decimal_places=2)
    category_id: int = Field(..., gt=0)

class ProductCreate(ProductBase):
    sku: str = Field(..., pattern=r"^[A-Z]{2,4}-\d{4}-\d{3}$")

class ProductUpdate(BaseModel):
    """All fields optional for partial updates."""
    name: str | None = Field(None, min_length=1, max_length=200)
    description: str | None = None
    price: Decimal | None = Field(None, gt=0, decimal_places=2)
    category_id: int | None = Field(None, gt=0)

class ProductResponse(ProductBase):
    id: int
    sku: str
    created_at: datetime
    is_active: bool

class ProductDetailResponse(ProductResponse):
    """Extended response with related data for single-item endpoints."""
    category: CategoryResponse
    variants: list[VariantResponse]
    average_rating: float | None
    review_count: int
```

### HTTP Status Codes

To use status codes correctly and consistently across all endpoints:

| Operation | Success Code | Description |
|-----------|-------------|-------------|
| GET (single) | 200 | Resource returned |
| GET (list) | 200 | Collection returned |
| POST (create) | 201 | Resource created |
| PATCH (update) | 200 | Resource updated and returned |
| PUT (replace) | 200 | Resource replaced and returned |
| DELETE | 204 | Resource deleted, no content |
| POST (action) | 200 or 202 | Action completed or accepted for processing |

Error codes to use:
- 400: Bad request (malformed syntax, invalid business logic)
- 401: Unauthenticated (missing or invalid credentials)
- 403: Forbidden (authenticated but insufficient permissions)
- 404: Resource not found
- 409: Conflict (duplicate resource, state conflict)
- 413: Payload too large
- 422: Validation error (Pydantic validation failures)
- 429: Rate limit exceeded
- 500: Internal server error (unhandled exceptions)

### Pagination

To implement offset-based pagination with a standard response wrapper:

```python
from typing import Generic, TypeVar
from pydantic import BaseModel, computed_field

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int

    @computed_field
    @property
    def has_next(self) -> bool:
        return self.page * self.page_size < self.total

    @computed_field
    @property
    def has_previous(self) -> bool:
        return self.page > 1

    @computed_field
    @property
    def total_pages(self) -> int:
        return (self.total + self.page_size - 1) // self.page_size
```

To implement cursor-based pagination for large datasets:

```python
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")

class CursorPage(BaseModel, Generic[T]):
    items: list[T]
    next_cursor: str | None
    previous_cursor: str | None
    has_more: bool

@router.get("/events", response_model=CursorPage[EventResponse])
async def list_events(
    cursor: str | None = Query(None, description="Pagination cursor"),
    limit: int = Query(20, ge=1, le=100),
    service: EventService = Depends(),
):
    return await service.list_with_cursor(cursor=cursor, limit=limit)
```

Cursor-based pagination avoids the performance problems of large offsets and provides stable pagination even when data changes between requests.

### Filtering and Sorting

To implement reusable filter and sort query parameters:

```python
from fastapi import Query
from enum import Enum

class SortOrder(str, Enum):
    asc = "asc"
    desc = "desc"

class ProductFilters:
    def __init__(
        self,
        category_id: int | None = Query(None, description="Filter by category"),
        min_price: Decimal | None = Query(None, ge=0, description="Minimum price"),
        max_price: Decimal | None = Query(None, ge=0, description="Maximum price"),
        is_active: bool | None = Query(None, description="Filter by active status"),
        search: str | None = Query(None, min_length=1, max_length=100, description="Search in name and description"),
        sort_by: str = Query("created_at", description="Sort field"),
        sort_order: SortOrder = Query(SortOrder.desc, description="Sort direction"),
    ):
        self.category_id = category_id
        self.min_price = min_price
        self.max_price = max_price
        self.is_active = is_active
        self.search = search
        self.sort_by = sort_by
        self.sort_order = sort_order

@router.get("/", response_model=PaginatedResponse[ProductResponse])
async def list_products(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    filters: ProductFilters = Depends(),
    service: ProductService = Depends(),
):
    return await service.list_filtered(
        page=page, page_size=page_size, filters=filters
    )
```

### API Versioning

To version APIs via URL prefix, organize routers by version:

```python
# app/main.py
from app.routers.v1 import users as users_v1, products as products_v1
from app.routers.v2 import users as users_v2

app.include_router(users_v1.router, prefix="/api/v1")
app.include_router(products_v1.router, prefix="/api/v1")
app.include_router(users_v2.router, prefix="/api/v2")
```

To version via header-based routing when URL versioning is not desired:

```python
from fastapi import Header, HTTPException

async def get_api_version(
    x_api_version: str = Header("1", alias="X-API-Version")
) -> int:
    try:
        version = int(x_api_version)
    except ValueError:
        raise HTTPException(400, "X-API-Version must be an integer")
    if version not in (1, 2):
        raise HTTPException(400, f"API version {version} not supported")
    return version
```

### Error Response Envelope

To standardize error responses across the API:

```python
class ErrorResponse(BaseModel):
    error: str = Field(..., description="Machine-readable error code")
    detail: str = Field(..., description="Human-readable error message")
    errors: list[FieldError] | None = Field(None, description="Field-level validation errors")

class FieldError(BaseModel):
    field: str
    message: str
    type: str
```

Document the error model on all endpoints:

```python
error_responses = {
    400: {"model": ErrorResponse, "description": "Bad request"},
    401: {"model": ErrorResponse, "description": "Not authenticated"},
    403: {"model": ErrorResponse, "description": "Insufficient permissions"},
    404: {"model": ErrorResponse, "description": "Resource not found"},
    422: {"model": ErrorResponse, "description": "Validation error"},
}

router = APIRouter(responses=error_responses)
```

### OpenAPI Documentation

To enrich the OpenAPI schema with detailed descriptions, tag groupings, and examples:

```python
app = FastAPI(
    title="Product Catalog API",
    description="""
Product Catalog API provides endpoints for managing products, categories,
and inventory.

## Authentication
All endpoints require a Bearer token in the Authorization header.

## Rate Limiting
API calls are limited to 100 requests per minute per API key.
    """,
    version="2.0.0",
    openapi_tags=[
        {
            "name": "Products",
            "description": "CRUD operations for product management",
        },
        {
            "name": "Categories",
            "description": "Product category management",
        },
        {
            "name": "System",
            "description": "Health checks and system information",
        },
    ],
    servers=[
        {"url": "https://api.example.com", "description": "Production"},
        {"url": "https://staging-api.example.com", "description": "Staging"},
    ],
)
```

To add operation-level documentation:

```python
@router.post(
    "/",
    response_model=ProductResponse,
    status_code=201,
    summary="Create a new product",
    description="""
Create a new product in the catalog. The SKU must be unique across
all products. The product is created in active status by default.
    """,
    response_description="The newly created product",
    responses={
        409: {
            "model": ErrorResponse,
            "description": "Product with this SKU already exists",
            "content": {
                "application/json": {
                    "example": {
                        "error": "CONFLICT",
                        "detail": "Product with sku 'WM-2024-001' already exists",
                    }
                }
            },
        }
    },
)
async def create_product(product_in: ProductCreate):
    ...
```

## References

- [references/api-patterns.md](references/api-patterns.md) - HATEOAS, cursor pagination, bulk operations, file uploads, streaming responses, and WebSocket patterns
- [references/api-security.md](references/api-security.md) - OAuth2 with scopes, JWT implementation, API key authentication, CORS, rate limiting, input validation, and security headers
