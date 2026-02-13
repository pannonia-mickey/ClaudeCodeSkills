# Architecture Patterns for FastAPI

This reference covers advanced architectural patterns for FastAPI applications: Hexagonal Architecture, CQRS, multi-module project organization, configuration management, and service communication.

---

## Hexagonal Architecture (Full Implementation)

Hexagonal Architecture (Ports & Adapters) isolates the application core from external systems. The core defines "ports" (interfaces); the outside world connects via "adapters" that implement those ports.

### Port Definitions

```python
# app/domain/ports/repository.py
from typing import Protocol, Generic, TypeVar, Sequence

T = TypeVar("T")

class ReadPort(Protocol[T]):
    async def get_by_id(self, id: int) -> T | None: ...
    async def list_all(self, skip: int = 0, limit: int = 20) -> Sequence[T]: ...
    async def count(self) -> int: ...

class WritePort(Protocol[T]):
    async def save(self, entity: T) -> T: ...
    async def delete(self, id: int) -> bool: ...

class OrderRepositoryPort(ReadPort["Order"], WritePort["Order"], Protocol):
    async def list_by_user(self, user_id: int, skip: int, limit: int) -> Sequence["Order"]: ...
    async def get_with_items(self, order_id: int) -> "Order | None": ...
```

```python
# app/domain/ports/notification.py
from typing import Protocol

class NotificationPort(Protocol):
    async def send_email(self, to: str, subject: str, body: str) -> None: ...
    async def send_sms(self, to: str, message: str) -> None: ...

class PaymentPort(Protocol):
    async def charge(self, amount: Decimal, currency: str, payment_method: str) -> "PaymentResult": ...
    async def refund(self, transaction_id: str, amount: Decimal | None = None) -> "RefundResult": ...
```

### Adapter Implementations

```python
# app/adapters/persistence/sqlalchemy_order_repo.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload
from app.domain.entities import Order, OrderItem
from app.models.order import OrderModel, OrderItemModel

class SqlAlchemyOrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, id: int) -> Order | None:
        result = await self.session.execute(
            select(OrderModel).where(OrderModel.id == id)
        )
        model = result.scalar_one_or_none()
        return self._to_domain(model) if model else None

    async def get_with_items(self, order_id: int) -> Order | None:
        result = await self.session.execute(
            select(OrderModel)
            .where(OrderModel.id == order_id)
            .options(selectinload(OrderModel.items))
        )
        model = result.scalar_one_or_none()
        return self._to_domain(model) if model else None

    async def list_by_user(
        self, user_id: int, skip: int = 0, limit: int = 20
    ) -> list[Order]:
        result = await self.session.execute(
            select(OrderModel)
            .where(OrderModel.user_id == user_id)
            .offset(skip)
            .limit(limit)
            .options(selectinload(OrderModel.items))
        )
        return [self._to_domain(m) for m in result.scalars().all()]

    async def list_all(self, skip: int = 0, limit: int = 20) -> list[Order]:
        result = await self.session.execute(
            select(OrderModel).offset(skip).limit(limit)
        )
        return [self._to_domain(m) for m in result.scalars().all()]

    async def count(self) -> int:
        result = await self.session.execute(
            select(func.count()).select_from(OrderModel)
        )
        return result.scalar_one()

    async def save(self, order: Order) -> Order:
        if order.id:
            model = await self.session.get(OrderModel, order.id)
            self._update_model(model, order)
        else:
            model = self._to_model(order)
            self.session.add(model)

        await self.session.flush()
        await self.session.refresh(model)
        return self._to_domain(model)

    async def delete(self, id: int) -> bool:
        model = await self.session.get(OrderModel, id)
        if not model:
            return False
        await self.session.delete(model)
        await self.session.flush()
        return True

    def _to_domain(self, model: OrderModel) -> Order:
        return Order(
            id=model.id,
            user_id=model.user_id,
            status=model.status,
            items=[
                OrderItem(
                    product_id=item.product_id,
                    product_name=item.product_name,
                    unit_price=Money(item.price),
                    quantity=item.quantity,
                )
                for item in (model.items or [])
            ],
            created_at=model.created_at,
        )

    def _to_model(self, entity: Order) -> OrderModel:
        return OrderModel(
            user_id=entity.user_id,
            status=entity.status,
            total=entity.total.amount,
        )

    def _update_model(self, model: OrderModel, entity: Order) -> None:
        model.status = entity.status
        model.total = entity.total.amount
```

```python
# app/adapters/notifications/email_sender.py
import httpx

class SendGridEmailSender:
    def __init__(self, api_key: str, from_email: str):
        self.api_key = api_key
        self.from_email = from_email

    async def send_email(self, to: str, subject: str, body: str) -> None:
        async with httpx.AsyncClient() as client:
            await client.post(
                "https://api.sendgrid.com/v3/mail/send",
                headers={"Authorization": f"Bearer {self.api_key}"},
                json={
                    "personalizations": [{"to": [{"email": to}]}],
                    "from": {"email": self.from_email},
                    "subject": subject,
                    "content": [{"type": "text/html", "value": body}],
                },
            )

    async def send_sms(self, to: str, message: str) -> None:
        raise NotImplementedError("SMS not supported by email sender")
```

### Dependency Wiring

```python
# app/dependencies.py
from functools import lru_cache
from app.config import get_settings
from app.domain.ports.repository import OrderRepositoryPort
from app.domain.ports.notification import NotificationPort, PaymentPort
from app.adapters.persistence.sqlalchemy_order_repo import SqlAlchemyOrderRepository
from app.adapters.notifications.email_sender import SendGridEmailSender
from app.adapters.payments.stripe_adapter import StripePaymentGateway

async def get_order_repo(
    session: AsyncSession = Depends(get_db),
) -> OrderRepositoryPort:
    return SqlAlchemyOrderRepository(session)

@lru_cache
def get_notification_service() -> NotificationPort:
    settings = get_settings()
    return SendGridEmailSender(
        api_key=settings.sendgrid_api_key.get_secret_value(),
        from_email=settings.from_email,
    )

@lru_cache
def get_payment_gateway() -> PaymentPort:
    settings = get_settings()
    return StripePaymentGateway(
        api_key=settings.stripe_secret_key.get_secret_value(),
    )
```

---

## CQRS (Command Query Responsibility Segregation)

CQRS separates read and write operations into distinct models. This is valuable when read and write patterns diverge significantly (e.g., complex aggregation queries for reads, transactional integrity for writes).

### Command Side (Writes)

```python
# app/commands/orders.py
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class PlaceOrderCommand:
    user_id: int
    items: list[dict]  # [{"product_id": 1, "quantity": 2}]

@dataclass
class CancelOrderCommand:
    order_id: int
    reason: str

@dataclass
class UpdateOrderStatusCommand:
    order_id: int
    new_status: str
```

```python
# app/commands/handlers.py
class OrderCommandHandler:
    def __init__(
        self,
        uow: UnitOfWork = Depends(),
        event_bus: EventBus = Depends(get_event_bus),
    ):
        self.uow = uow
        self.event_bus = event_bus

    async def handle_place_order(self, cmd: PlaceOrderCommand) -> int:
        async with self.uow:
            order = Order(user_id=cmd.user_id, status="pending")
            for item_data in cmd.items:
                product = await self.uow.products.get_by_id(item_data["product_id"])
                if not product or product.stock < item_data["quantity"]:
                    raise InsufficientStockError(item_data["product_id"])
                order.add_item(OrderItem(
                    product_id=product.id,
                    product_name=product.name,
                    unit_price=Money(product.price),
                    quantity=item_data["quantity"],
                ))
                await self.uow.products.decrement_stock(product.id, item_data["quantity"])

            order.submit()
            saved = await self.uow.orders.save(order)
            await self.uow.commit()

        await self.event_bus.publish(OrderPlaced(
            order_id=saved.id, user_id=cmd.user_id, total=str(saved.total.amount)
        ))
        return saved.id

    async def handle_cancel_order(self, cmd: CancelOrderCommand) -> None:
        async with self.uow:
            order = await self.uow.orders.get_with_items(cmd.order_id)
            if not order:
                raise NotFoundError("Order", cmd.order_id)
            order.cancel()
            await self.uow.orders.save(order)
            await self.uow.commit()

        await self.event_bus.publish(OrderCancelled(
            order_id=cmd.order_id, reason=cmd.reason
        ))
```

### Query Side (Reads)

```python
# app/queries/orders.py
from dataclasses import dataclass

@dataclass
class GetOrderQuery:
    order_id: int

@dataclass
class ListUserOrdersQuery:
    user_id: int
    page: int = 1
    page_size: int = 20

@dataclass
class OrderSummaryQuery:
    """Optimized read model for dashboard."""
    user_id: int
    start_date: date | None = None
    end_date: date | None = None
```

```python
# app/queries/handlers.py
class OrderQueryHandler:
    def __init__(self, read_db: AsyncSession = Depends(get_read_db)):
        self.db = read_db

    async def get_order(self, query: GetOrderQuery) -> OrderDetailView:
        result = await self.db.execute(
            select(OrderModel)
            .where(OrderModel.id == query.order_id)
            .options(
                selectinload(OrderModel.items).joinedload(OrderItemModel.product),
                joinedload(OrderModel.user),
            )
        )
        order = result.scalar_one_or_none()
        if not order:
            raise NotFoundError("Order", query.order_id)
        return OrderDetailView.model_validate(order)

    async def list_user_orders(self, query: ListUserOrdersQuery) -> PaginatedResponse:
        count_stmt = (
            select(func.count())
            .select_from(OrderModel)
            .where(OrderModel.user_id == query.user_id)
        )
        total = (await self.db.execute(count_stmt)).scalar_one()

        offset = (query.page - 1) * query.page_size
        items_stmt = (
            select(OrderModel)
            .where(OrderModel.user_id == query.user_id)
            .order_by(OrderModel.created_at.desc())
            .offset(offset)
            .limit(query.page_size)
        )
        items = (await self.db.execute(items_stmt)).scalars().all()

        return PaginatedResponse(
            items=[OrderListView.model_validate(o) for o in items],
            total=total,
            page=query.page,
            page_size=query.page_size,
        )

    async def get_order_summary(self, query: OrderSummaryQuery) -> OrderSummaryView:
        """Optimized aggregation query directly against DB - no domain model needed."""
        stmt = (
            select(
                func.count(OrderModel.id).label("total_orders"),
                func.sum(OrderModel.total).label("total_spent"),
                func.avg(OrderModel.total).label("avg_order_value"),
            )
            .where(OrderModel.user_id == query.user_id)
        )
        if query.start_date:
            stmt = stmt.where(OrderModel.created_at >= query.start_date)
        if query.end_date:
            stmt = stmt.where(OrderModel.created_at <= query.end_date)

        result = (await self.db.execute(stmt)).one()
        return OrderSummaryView(
            total_orders=result.total_orders or 0,
            total_spent=result.total_spent or Decimal("0"),
            avg_order_value=result.avg_order_value or Decimal("0"),
        )
```

### CQRS Router Integration

```python
# app/routers/orders.py
router = APIRouter(prefix="/orders", tags=["Orders"])

@router.post("/", status_code=201)
async def create_order(
    order_in: OrderCreate,
    handler: OrderCommandHandler = Depends(),
):
    order_id = await handler.handle_place_order(
        PlaceOrderCommand(user_id=order_in.user_id, items=order_in.items)
    )
    return {"id": order_id}

@router.get("/{order_id}", response_model=OrderDetailView)
async def get_order(
    order_id: int,
    handler: OrderQueryHandler = Depends(),
):
    return await handler.get_order(GetOrderQuery(order_id=order_id))

@router.get("/", response_model=PaginatedResponse[OrderListView])
async def list_orders(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    user: User = Depends(get_current_user),
    handler: OrderQueryHandler = Depends(),
):
    return await handler.list_user_orders(
        ListUserOrdersQuery(user_id=user.id, page=page, page_size=page_size)
    )
```

---

## Multi-Module Project Organization

For large applications, organize by business domain (vertical slices) rather than by technical layer:

```
app/
    shared/                     # Cross-cutting shared code
        dependencies.py         # Common dependencies (get_db, get_current_user)
        exceptions.py           # Base exception hierarchy
        schemas.py              # Shared schemas (PaginatedResponse, ErrorResponse)
        middleware.py            # Security headers, CORS, request ID
    modules/
        users/                  # User domain module
            router.py
            schemas.py
            service.py
            repository.py
            models.py
        products/               # Product domain module
            router.py
            schemas.py
            service.py
            repository.py
            models.py
        orders/                 # Order domain module
            router.py
            schemas.py
            service.py
            repository.py
            models.py
            events.py           # Domain events specific to orders
    main.py
    config.py
```

```python
# app/main.py - auto-discover and register modules
import importlib
import pkgutil
from pathlib import Path

def _register_modules(app: FastAPI) -> None:
    """Auto-discover modules and register their routers."""
    modules_path = Path(__file__).parent / "modules"
    for module_info in pkgutil.iter_modules([str(modules_path)]):
        module = importlib.import_module(f"app.modules.{module_info.name}.router")
        if hasattr(module, "router"):
            app.include_router(module.router, prefix="/api/v1")
```

Each module is self-contained: it defines its own router, schemas, service, repository, and models. Modules communicate through well-defined service interfaces, not by importing each other's internals.

### Inter-Module Communication

```python
# Modules depend on each other through abstract interfaces, not concrete imports

# app/modules/orders/service.py
class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository = Depends(),
        product_service: "ProductServicePort" = Depends(get_product_service),
        notification_service: "NotificationPort" = Depends(get_notification_service),
    ):
        self.order_repo = order_repo
        self.product_service = product_service
        self.notification_service = notification_service

    async def place_order(self, data: OrderCreate) -> Order:
        # Call product module through its port (interface)
        for item in data.items:
            available = await self.product_service.check_stock(
                item.product_id, item.quantity
            )
            if not available:
                raise InsufficientStockError(item.product_id)

        order = await self.order_repo.create(data)

        # Notify through notification port
        await self.notification_service.send_email(
            to=data.user_email,
            subject="Order Confirmation",
            body=f"Your order #{order.id} has been placed.",
        )

        return order
```

---

## Configuration Management

### Environment-Specific Configuration

```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, SecretStr, computed_field
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="APP_",
        case_sensitive=False,
    )

    # Application
    app_name: str = "MyService"
    environment: str = "production"
    debug: bool = False
    log_level: str = "INFO"

    # Database
    database_url: SecretStr
    db_pool_size: int = Field(default=5, ge=1, le=50)

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Security
    secret_key: SecretStr
    access_token_expire_minutes: int = 30
    allowed_origins: list[str] = ["http://localhost:3000"]

    # External services
    sendgrid_api_key: SecretStr = SecretStr("")
    stripe_secret_key: SecretStr = SecretStr("")

    @computed_field
    @property
    def is_production(self) -> bool:
        return self.environment == "production"

    @computed_field
    @property
    def is_testing(self) -> bool:
        return self.environment == "testing"

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

### Feature Flags via Configuration

```python
class FeatureFlags(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="FEATURE_")

    enable_new_checkout: bool = False
    enable_ai_recommendations: bool = False
    max_bulk_import_size: int = 1000

@lru_cache
def get_feature_flags() -> FeatureFlags:
    return FeatureFlags()

# Usage in endpoint
@router.post("/checkout")
async def checkout(
    data: CheckoutRequest,
    features: FeatureFlags = Depends(get_feature_flags),
    service: CheckoutService = Depends(),
):
    if features.enable_new_checkout:
        return await service.new_checkout(data)
    return await service.legacy_checkout(data)
```

---

## Service Communication Patterns

### Internal Service Clients

When breaking a monolith into services, abstract inter-service calls behind typed clients:

```python
# app/clients/product_client.py
from typing import Protocol

class ProductServiceClient(Protocol):
    async def get_product(self, product_id: int) -> ProductDTO: ...
    async def check_stock(self, product_id: int, quantity: int) -> bool: ...
    async def reserve_stock(self, product_id: int, quantity: int) -> str: ...

class HttpProductClient:
    """HTTP client for the Product microservice."""

    def __init__(self, base_url: str, timeout: float = 10.0):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout

    async def get_product(self, product_id: int) -> ProductDTO:
        async with httpx.AsyncClient(timeout=self.timeout) as client:
            response = await client.get(f"{self.base_url}/api/v1/products/{product_id}")
            response.raise_for_status()
            return ProductDTO(**response.json())

    async def check_stock(self, product_id: int, quantity: int) -> bool:
        async with httpx.AsyncClient(timeout=self.timeout) as client:
            response = await client.get(
                f"{self.base_url}/api/v1/products/{product_id}/stock",
                params={"quantity": quantity},
            )
            return response.json()["available"]

    async def reserve_stock(self, product_id: int, quantity: int) -> str:
        async with httpx.AsyncClient(timeout=self.timeout) as client:
            response = await client.post(
                f"{self.base_url}/api/v1/products/{product_id}/reserve",
                json={"quantity": quantity},
            )
            response.raise_for_status()
            return response.json()["reservation_id"]

class LocalProductClient:
    """In-process client when products are in the same service."""

    def __init__(self, repo: ProductRepository):
        self.repo = repo

    async def get_product(self, product_id: int) -> ProductDTO:
        product = await self.repo.get_by_id(product_id)
        if not product:
            raise NotFoundError("Product", product_id)
        return ProductDTO.from_model(product)

    async def check_stock(self, product_id: int, quantity: int) -> bool:
        product = await self.repo.get_by_id(product_id)
        return product is not None and product.stock >= quantity
```

This dual-client pattern allows gradual extraction: start with `LocalProductClient`, switch to `HttpProductClient` when the product service is deployed independently.
