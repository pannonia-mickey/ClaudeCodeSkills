---
name: FastAPI Architecture
description: This skill should be used when the user asks about "FastAPI architecture", "FastAPI project structure", "layered architecture", "clean architecture FastAPI", "hexagonal architecture", "DDD FastAPI", "domain driven design", "FastAPI microservice", "CQRS FastAPI", "event driven FastAPI", or "FastAPI monorepo". It covers layered architecture enforcement, Clean Architecture and Hexagonal Architecture adapted for FastAPI, Domain-Driven Design tactical patterns, event-driven design, CQRS, service communication, and multi-module project organization. Use this skill for any task involving architectural decisions, project structuring, layer separation, domain modeling, or scaling a FastAPI application beyond simple CRUD.
---

### Layered Architecture

To enforce clean separation of concerns, organize a FastAPI application into strict layers with a unidirectional dependency rule: each layer depends only on the layer directly below it.

```
Routers (HTTP/Transport) -> Services (Business Logic) -> Repositories (Data Access) -> Models (Domain)
```

```
app/
    routers/          # HTTP layer: endpoints, request parsing, response serialization
    schemas/          # Data Transfer Objects: Pydantic models for API boundaries
    services/         # Business logic: orchestration, rules, validation
    repositories/     # Data access: database queries, external API calls
    models/           # Domain models: SQLAlchemy ORM or pure domain entities
    domain/           # Domain layer: value objects, domain events, business rules
    dependencies.py   # FastAPI Depends() wiring
    exceptions.py     # Application exception hierarchy
    config.py         # Settings via pydantic-settings
    main.py           # App factory, middleware, lifespan
```

Rules to enforce:
- Routers never import from repositories or models directly. They depend only on schemas and services.
- Services never import from routers. They depend on repositories and domain logic.
- Repositories never contain business logic. They translate between domain models and persistence.
- Schemas never inherit from ORM models. Keep API contracts independent from storage.

### Thin Routers, Rich Services

Endpoint handlers should be thin wrappers that delegate to service methods. All business logic, validation beyond schema parsing, and orchestration belong in the service layer.

```python
# app/routers/orders.py - THIN: parse, delegate, return
from fastapi import APIRouter, Depends, status
from app.schemas.orders import OrderCreate, OrderResponse
from app.services.order_service import OrderService

router = APIRouter(prefix="/orders", tags=["Orders"])

@router.post("/", response_model=OrderResponse, status_code=status.HTTP_201_CREATED)
async def create_order(
    order_in: OrderCreate,
    service: OrderService = Depends(),
):
    return await service.place_order(order_in)
```

```python
# app/services/order_service.py - RICH: all business logic here
from fastapi import Depends
from app.repositories.order_repo import OrderRepository
from app.repositories.product_repo import ProductRepository
from app.schemas.orders import OrderCreate
from app.exceptions import InsufficientStockError, NotFoundError

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository = Depends(),
        product_repo: ProductRepository = Depends(),
    ):
        self.order_repo = order_repo
        self.product_repo = product_repo

    async def place_order(self, data: OrderCreate) -> Order:
        # Validate stock availability
        for item in data.items:
            product = await self.product_repo.get_by_id(item.product_id)
            if not product:
                raise NotFoundError("Product", item.product_id)
            if product.stock < item.quantity:
                raise InsufficientStockError(product.name, item.quantity, product.stock)

        # Reserve stock and create order atomically
        total = await self._calculate_total(data.items)
        order = await self.order_repo.create_with_items(
            user_id=data.user_id,
            items=data.items,
            total=total,
        )

        # Decrement stock
        for item in data.items:
            await self.product_repo.decrement_stock(item.product_id, item.quantity)

        return order

    async def _calculate_total(self, items: list) -> Decimal:
        total = Decimal("0")
        for item in items:
            product = await self.product_repo.get_by_id(item.product_id)
            total += product.price * item.quantity
        return total
```

### Clean Architecture (Ports and Adapters)

To decouple business logic from framework and infrastructure, define abstract interfaces (ports) in the domain layer and implement them as adapters in the infrastructure layer.

```
app/
    domain/
        ports/
            repository.py        # Abstract repository interfaces
            notification.py      # Abstract notification interfaces
            payment.py           # Abstract payment interfaces
        entities.py              # Pure domain entities (no ORM dependency)
        value_objects.py         # Immutable value types
        services.py              # Domain services (pure business rules)
    adapters/
        persistence/
            sqlalchemy_repo.py   # SQLAlchemy repository implementation
        notifications/
            email_sender.py      # SMTP implementation
            sms_sender.py        # SMS implementation
        payments/
            stripe_adapter.py    # Stripe payment implementation
    routers/                     # HTTP adapter (FastAPI endpoints)
    dependencies.py              # Wiring: binds ports to adapters
```

```python
# app/domain/ports/repository.py - Port (abstract interface)
from typing import Protocol, TypeVar

T = TypeVar("T")

class OrderRepositoryPort(Protocol):
    async def get_by_id(self, order_id: int) -> Order | None: ...
    async def save(self, order: Order) -> Order: ...
    async def list_by_user(self, user_id: int, skip: int, limit: int) -> list[Order]: ...

class PaymentGatewayPort(Protocol):
    async def charge(self, amount: Decimal, currency: str, token: str) -> PaymentResult: ...
    async def refund(self, transaction_id: str, amount: Decimal) -> RefundResult: ...
```

```python
# app/adapters/persistence/sqlalchemy_order_repo.py - Adapter (concrete implementation)
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from fastapi import Depends
from app.dependencies import get_db

class SqlAlchemyOrderRepository:
    def __init__(self, session: AsyncSession = Depends(get_db)):
        self.session = session

    async def get_by_id(self, order_id: int) -> Order | None:
        result = await self.session.execute(
            select(OrderModel).where(OrderModel.id == order_id)
        )
        row = result.scalar_one_or_none()
        return row.to_domain() if row else None

    async def save(self, order: Order) -> Order:
        model = OrderModel.from_domain(order)
        self.session.add(model)
        await self.session.flush()
        await self.session.refresh(model)
        return model.to_domain()
```

```python
# app/dependencies.py - Wiring ports to adapters
from app.domain.ports.repository import OrderRepositoryPort
from app.adapters.persistence.sqlalchemy_order_repo import SqlAlchemyOrderRepository
from app.adapters.payments.stripe_adapter import StripePaymentGateway

async def get_order_repository() -> OrderRepositoryPort:
    return SqlAlchemyOrderRepository()

async def get_payment_gateway() -> PaymentGatewayPort:
    return StripePaymentGateway()
```

The key benefit: domain services depend on ports (abstractions), not on SQLAlchemy, Stripe, or any framework. Swapping databases or payment providers requires changing only the adapter and the wiring.

### Domain-Driven Design Tactical Patterns

To model complex business domains, apply DDD building blocks within the domain layer.

**Value Objects** - Immutable types defined by their attributes, not identity:

```python
# app/domain/value_objects.py
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str = "USD"

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Money amount cannot be negative")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"Cannot add {self.currency} and {other.currency}")
        return Money(amount=self.amount + other.amount, currency=self.currency)

    def multiply(self, factor: int) -> "Money":
        return Money(amount=self.amount * factor, currency=self.currency)

@dataclass(frozen=True)
class Address:
    street: str
    city: str
    state: str
    zip_code: str
    country: str = "US"

@dataclass(frozen=True)
class EmailAddress:
    value: str

    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError(f"Invalid email: {self.value}")
```

**Entities and Aggregates** - Objects with identity and lifecycle:

```python
# app/domain/entities.py
from dataclasses import dataclass, field
from datetime import datetime
from app.domain.value_objects import Money

@dataclass
class OrderItem:
    product_id: int
    product_name: str
    unit_price: Money
    quantity: int

    @property
    def subtotal(self) -> Money:
        return self.unit_price.multiply(self.quantity)

@dataclass
class Order:
    """Aggregate root: all modifications to order items go through Order."""
    id: int | None = None
    user_id: int = 0
    items: list[OrderItem] = field(default_factory=list)
    status: str = "draft"
    created_at: datetime = field(default_factory=datetime.utcnow)

    @property
    def total(self) -> Money:
        if not self.items:
            return Money(Decimal("0"))
        result = self.items[0].subtotal
        for item in self.items[1:]:
            result = result.add(item.subtotal)
        return result

    def add_item(self, item: OrderItem) -> None:
        if self.status != "draft":
            raise DomainError("Cannot add items to a non-draft order")
        existing = next((i for i in self.items if i.product_id == item.product_id), None)
        if existing:
            raise DomainError(f"Product {item.product_id} already in order")
        self.items.append(item)

    def submit(self) -> None:
        if not self.items:
            raise DomainError("Cannot submit an empty order")
        if self.status != "draft":
            raise DomainError(f"Cannot submit order in '{self.status}' status")
        self.status = "pending"

    def cancel(self) -> None:
        if self.status in ("shipped", "delivered", "cancelled"):
            raise DomainError(f"Cannot cancel order in '{self.status}' status")
        self.status = "cancelled"
```

**Domain Services** - Operations that do not belong to a single entity:

```python
# app/domain/services.py
from app.domain.entities import Order
from app.domain.value_objects import Money
from decimal import Decimal

class PricingService:
    """Calculates pricing with discounts, taxes, and promotions."""

    def calculate_order_total(
        self, order: Order, discount_pct: Decimal = Decimal("0")
    ) -> Money:
        subtotal = order.total
        discount = Money(subtotal.amount * discount_pct / 100, subtotal.currency)
        return Money(subtotal.amount - discount.amount, subtotal.currency)
```

### Event-Driven Patterns

To decouple components that react to state changes, implement in-process domain events using a simple event bus.

```python
# app/domain/events.py
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any

@dataclass
class DomainEvent:
    occurred_at: datetime = field(default_factory=datetime.utcnow)

@dataclass
class OrderPlaced(DomainEvent):
    order_id: int = 0
    user_id: int = 0
    total: str = ""

@dataclass
class OrderCancelled(DomainEvent):
    order_id: int = 0
    user_id: int = 0
    reason: str = ""

@dataclass
class PaymentProcessed(DomainEvent):
    order_id: int = 0
    transaction_id: str = ""
    amount: str = ""
```

```python
# app/infrastructure/event_bus.py
from typing import Callable, Type
from collections import defaultdict
import asyncio

class EventBus:
    def __init__(self):
        self._handlers: dict[Type, list[Callable]] = defaultdict(list)

    def subscribe(self, event_type: Type, handler: Callable) -> None:
        self._handlers[event_type].append(handler)

    async def publish(self, event) -> None:
        handlers = self._handlers.get(type(event), [])
        await asyncio.gather(*(handler(event) for handler in handlers))

# Singleton instance
event_bus = EventBus()
```

```python
# app/services/handlers.py - Event handlers
from app.domain.events import OrderPlaced, OrderCancelled

async def send_order_confirmation(event: OrderPlaced) -> None:
    await email_service.send_template(
        template="order_confirmation",
        to_user_id=event.user_id,
        context={"order_id": event.order_id, "total": event.total},
    )

async def update_analytics(event: OrderPlaced) -> None:
    await analytics_service.track("order_placed", {
        "order_id": event.order_id,
        "total": event.total,
    })

async def restore_inventory(event: OrderCancelled) -> None:
    await inventory_service.restore_stock(event.order_id)

# Wire handlers at startup
def register_event_handlers(bus: EventBus) -> None:
    bus.subscribe(OrderPlaced, send_order_confirmation)
    bus.subscribe(OrderPlaced, update_analytics)
    bus.subscribe(OrderCancelled, restore_inventory)
```

### Application Factory Pattern

To configure the FastAPI application in a testable, composable way, use a factory function instead of a module-level app instance.

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.config import get_settings
from app.infrastructure.event_bus import event_bus
from app.services.handlers import register_event_handlers

def create_app(settings=None) -> FastAPI:
    settings = settings or get_settings()

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        # Startup
        app.state.db_pool = await create_async_pool(settings.database_url)
        register_event_handlers(event_bus)
        yield
        # Shutdown
        await app.state.db_pool.dispose()

    app = FastAPI(
        title=settings.app_name,
        version="1.0.0",
        lifespan=lifespan,
        docs_url="/docs" if settings.debug else None,
        redoc_url=None,
    )

    _register_middleware(app, settings)
    _register_routers(app)
    _register_exception_handlers(app)

    return app

def _register_middleware(app: FastAPI, settings) -> None:
    from fastapi.middleware.cors import CORSMiddleware
    from app.middleware import SecurityHeadersMiddleware, RequestContextMiddleware

    app.add_middleware(RequestContextMiddleware)
    app.add_middleware(SecurityHeadersMiddleware)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.allowed_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

def _register_routers(app: FastAPI) -> None:
    from app.routers import users, products, orders, health

    app.include_router(health.router)
    app.include_router(users.router, prefix="/api/v1")
    app.include_router(products.router, prefix="/api/v1")
    app.include_router(orders.router, prefix="/api/v1")

def _register_exception_handlers(app: FastAPI) -> None:
    from app.exceptions import AppError, app_error_handler
    app.add_exception_handler(AppError, app_error_handler)

# For uvicorn
app = create_app()
```

The factory pattern allows creating multiple app instances with different configurations, which is essential for testing, staging, and multi-tenant deployments.

### Service Layer with Unit of Work

To coordinate writes across multiple repositories atomically, pair services with the Unit of Work pattern.

```python
# app/services/order_service.py
from app.infrastructure.uow import UnitOfWork
from app.domain.events import OrderPlaced
from app.infrastructure.event_bus import event_bus

class OrderService:
    def __init__(self, uow: UnitOfWork = Depends()):
        self.uow = uow

    async def place_order(self, data: OrderCreate) -> OrderResponse:
        async with self.uow:
            # All repository operations share the same transaction
            user = await self.uow.users.get_by_id(data.user_id)
            if not user:
                raise NotFoundError("User", data.user_id)

            order = await self.uow.orders.create(
                user_id=data.user_id, status="pending"
            )

            for item in data.items:
                product = await self.uow.products.get_by_id(item.product_id)
                if product.stock < item.quantity:
                    raise InsufficientStockError(product.name)
                await self.uow.products.decrement_stock(item.product_id, item.quantity)
                await self.uow.order_items.create(
                    order_id=order.id,
                    product_id=item.product_id,
                    quantity=item.quantity,
                    price=product.price,
                )

            await self.uow.commit()

        # Publish event AFTER commit succeeds
        await event_bus.publish(OrderPlaced(
            order_id=order.id,
            user_id=data.user_id,
            total=str(order.total),
        ))

        return OrderResponse.model_validate(order)
```

## References

- [references/architecture-patterns.md](references/architecture-patterns.md) - Hexagonal Architecture implementation, CQRS pattern, multi-module project organization, and configuration management
- [references/domain-modeling.md](references/domain-modeling.md) - DDD aggregate design rules, domain event patterns, bounded context mapping, and ORM-to-domain mapping strategies
