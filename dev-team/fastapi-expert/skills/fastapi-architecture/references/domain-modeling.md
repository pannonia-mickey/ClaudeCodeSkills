# Domain Modeling for FastAPI

This reference covers Domain-Driven Design tactical patterns applied to FastAPI applications: aggregate design, domain event implementation, bounded context mapping, and ORM-to-domain mapping strategies.

---

## Aggregate Design Rules

Aggregates are clusters of domain objects treated as a single unit for data changes. The aggregate root is the only entry point for modifications.

### Rule 1: Protect Invariants Inside the Aggregate

All business rules that must be consistent are enforced by the aggregate root. External code never modifies child entities directly.

```python
# app/domain/aggregates/order.py
from dataclasses import dataclass, field
from decimal import Decimal
from datetime import datetime
from app.domain.value_objects import Money
from app.domain.events import DomainEvent, OrderPlaced, ItemAdded

@dataclass
class OrderItem:
    product_id: int
    product_name: str
    unit_price: Money
    quantity: int

    @property
    def line_total(self) -> Money:
        return self.unit_price.multiply(self.quantity)

@dataclass
class Order:
    """Aggregate root. All modifications go through this class."""
    id: int | None = None
    user_id: int = 0
    status: str = "draft"
    items: list[OrderItem] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.utcnow)
    _events: list[DomainEvent] = field(default_factory=list, repr=False)

    MAX_ITEMS = 50
    MAX_ORDER_VALUE = Money(Decimal("10000"))

    @property
    def total(self) -> Money:
        if not self.items:
            return Money(Decimal("0"))
        result = self.items[0].line_total
        for item in self.items[1:]:
            result = result.add(item.line_total)
        return result

    @property
    def item_count(self) -> int:
        return sum(item.quantity for item in self.items)

    def add_item(self, item: OrderItem) -> None:
        """Business rule: can only add to draft orders, no duplicates, within limits."""
        self._assert_draft()
        if any(i.product_id == item.product_id for i in self.items):
            raise DomainError(f"Product {item.product_id} already in order. Update quantity instead.")
        if len(self.items) >= self.MAX_ITEMS:
            raise DomainError(f"Order cannot exceed {self.MAX_ITEMS} line items")

        projected_total = self.total.add(item.line_total)
        if projected_total.amount > self.MAX_ORDER_VALUE.amount:
            raise DomainError(f"Order total would exceed {self.MAX_ORDER_VALUE.amount}")

        self.items.append(item)
        self._events.append(ItemAdded(order_id=self.id, product_id=item.product_id))

    def remove_item(self, product_id: int) -> None:
        self._assert_draft()
        self.items = [i for i in self.items if i.product_id != product_id]

    def update_item_quantity(self, product_id: int, new_quantity: int) -> None:
        self._assert_draft()
        if new_quantity < 1:
            raise DomainError("Quantity must be at least 1")
        for item in self.items:
            if item.product_id == product_id:
                item.quantity = new_quantity
                return
        raise DomainError(f"Product {product_id} not in order")

    def submit(self) -> None:
        self._assert_draft()
        if not self.items:
            raise DomainError("Cannot submit an empty order")
        self.status = "pending"
        self._events.append(OrderPlaced(
            order_id=self.id, user_id=self.user_id, total=str(self.total.amount)
        ))

    def confirm(self) -> None:
        if self.status != "pending":
            raise DomainError(f"Cannot confirm order in '{self.status}' status")
        self.status = "confirmed"

    def ship(self, tracking_number: str) -> None:
        if self.status != "confirmed":
            raise DomainError(f"Cannot ship order in '{self.status}' status")
        self.status = "shipped"

    def cancel(self, reason: str = "") -> None:
        non_cancellable = {"shipped", "delivered", "cancelled"}
        if self.status in non_cancellable:
            raise DomainError(f"Cannot cancel order in '{self.status}' status")
        self.status = "cancelled"

    def collect_events(self) -> list[DomainEvent]:
        events = self._events.copy()
        self._events.clear()
        return events

    def _assert_draft(self) -> None:
        if self.status != "draft":
            raise DomainError(f"Operation not allowed on order in '{self.status}' status")
```

### Rule 2: Reference Other Aggregates by ID

Aggregates reference each other by identity (ID), not by direct object reference. This prevents aggregates from becoming entangled.

```python
# GOOD: Reference by ID
@dataclass
class Order:
    user_id: int          # References User aggregate by ID
    items: list[OrderItem]  # OrderItem belongs to this aggregate

# BAD: Direct reference creates coupling
@dataclass
class Order:
    user: User              # Direct reference couples Order to User
    items: list[OrderItem]
```

### Rule 3: One Transaction Per Aggregate

Each command modifies exactly one aggregate and commits it. Cross-aggregate consistency is achieved through eventual consistency via domain events.

```python
# GOOD: Modify one aggregate per transaction
async def place_order(self, cmd: PlaceOrderCommand) -> int:
    async with self.uow:
        order = Order(user_id=cmd.user_id)
        for item in cmd.items:
            order.add_item(item)
        order.submit()
        saved = await self.uow.orders.save(order)
        await self.uow.commit()

    # Cross-aggregate effects via events (eventual consistency)
    for event in saved.collect_events():
        await self.event_bus.publish(event)
    return saved.id

# BAD: Modifying multiple aggregates in one transaction
async def place_order(self, cmd: PlaceOrderCommand) -> int:
    async with self.uow:
        order = Order(user_id=cmd.user_id)
        for item in cmd.items:
            order.add_item(item)
            # Modifying Product aggregate inside Order transaction
            product = await self.uow.products.get_by_id(item.product_id)
            product.stock -= item.quantity  # Violates aggregate boundary
        await self.uow.commit()
```

---

## Domain Event Patterns

### Event Collection on Aggregates

Aggregates collect events during business operations. Events are dispatched after the aggregate is persisted.

```python
# app/domain/events.py
from dataclasses import dataclass, field
from datetime import datetime

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
class ItemAdded(DomainEvent):
    order_id: int | None = None
    product_id: int = 0

@dataclass
class StockReserved(DomainEvent):
    product_id: int = 0
    quantity: int = 0
    reservation_id: str = ""

@dataclass
class PaymentCaptured(DomainEvent):
    order_id: int = 0
    transaction_id: str = ""
    amount: str = ""
```

### Event Bus with Async Handlers

```python
# app/infrastructure/event_bus.py
from typing import Callable, Type, Any
from collections import defaultdict
import asyncio
import structlog

logger = structlog.get_logger()

class EventBus:
    def __init__(self):
        self._handlers: dict[Type, list[Callable]] = defaultdict(list)

    def subscribe(self, event_type: Type, handler: Callable) -> None:
        self._handlers[event_type].append(handler)

    async def publish(self, event: Any) -> None:
        handlers = self._handlers.get(type(event), [])
        event_name = type(event).__name__

        for handler in handlers:
            handler_name = handler.__name__
            try:
                await handler(event)
                logger.info("event_handled", event=event_name, handler=handler_name)
            except Exception:
                logger.exception(
                    "event_handler_failed",
                    event=event_name,
                    handler=handler_name,
                )
                # Handler failure should not break the caller
                # In production, route to a dead letter queue

    async def publish_all(self, events: list) -> None:
        for event in events:
            await self.publish(event)
```

### Outbox Pattern for Reliable Event Delivery

To guarantee events are published even if the application crashes after commit, store events in an outbox table within the same transaction.

```python
# app/models/outbox.py
from sqlalchemy import String, JSON
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime

class OutboxMessage(Base):
    __tablename__ = "outbox_messages"

    id: Mapped[int] = mapped_column(primary_key=True)
    event_type: Mapped[str] = mapped_column(String(100))
    payload: Mapped[dict] = mapped_column(JSON)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    published_at: Mapped[datetime | None] = mapped_column(nullable=True)
```

```python
# app/infrastructure/outbox.py
import json
from dataclasses import asdict

class OutboxPublisher:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def store_event(self, event: DomainEvent) -> None:
        """Store event in the outbox table (same transaction as the aggregate)."""
        outbox_msg = OutboxMessage(
            event_type=type(event).__name__,
            payload=asdict(event),
        )
        self.session.add(outbox_msg)

class OutboxProcessor:
    """Background worker that reads the outbox and publishes to the event bus."""

    def __init__(self, session_factory, event_bus: EventBus):
        self.session_factory = session_factory
        self.event_bus = event_bus

    async def process_pending(self, batch_size: int = 100) -> int:
        async with self.session_factory() as session:
            result = await session.execute(
                select(OutboxMessage)
                .where(OutboxMessage.published_at.is_(None))
                .order_by(OutboxMessage.created_at)
                .limit(batch_size)
            )
            messages = result.scalars().all()

            for msg in messages:
                event_class = EVENT_REGISTRY.get(msg.event_type)
                if event_class:
                    event = event_class(**msg.payload)
                    await self.event_bus.publish(event)
                msg.published_at = datetime.utcnow()

            await session.commit()
            return len(messages)
```

---

## Bounded Context Mapping

When the application grows, different parts of the domain have different models for the same concept. A "User" in the authentication context differs from a "Customer" in the ordering context.

### Anti-Corruption Layer

When integrating with another context (or external service), translate between models at the boundary:

```python
# app/modules/orders/acl.py (Anti-Corruption Layer)
"""Translates between the User context and the Order context."""

from app.modules.users.service import UserService
from app.modules.orders.domain import Customer

class UserToCustomerTranslator:
    def __init__(self, user_service: UserService):
        self.user_service = user_service

    async def get_customer(self, user_id: int) -> Customer:
        """Translate User (auth context) to Customer (order context)."""
        user = await self.user_service.get_by_id(user_id)
        if not user:
            raise NotFoundError("User", user_id)
        return Customer(
            id=user.id,
            name=user.full_name,
            email=user.email,
            shipping_address=user.default_address,
        )
```

### Context Map

Document how bounded contexts relate to each other:

```python
# Context Map for an E-Commerce System
#
# [Identity Context] --- Customer/Supplier ---> [Order Context]
#   - User, Role, Session                        - Customer, Order, OrderItem
#   - Owns authentication and authorization       - Consumes user data via ACL
#
# [Product Catalog Context] --- Conformist ---> [Order Context]
#   - Product, Category, Variant                  - ProductSnapshot in OrderItem
#   - Owns product definitions                    - Copies price at time of order
#
# [Payment Context] --- Partnership ---> [Order Context]
#   - Transaction, PaymentMethod                  - Order triggers payment
#   - Owns payment processing                     - Payment confirms order
#
# [Inventory Context] --- Separate Ways ---> [Product Catalog Context]
#   - Stock, Warehouse, Reservation               - Separate stock management
#   - Owns stock levels                            - Product catalog is read-only
```

---

## ORM-to-Domain Mapping

### Mapping Between SQLAlchemy Models and Domain Entities

Keep ORM models in the infrastructure layer and domain entities in the domain layer. Map between them explicitly.

```python
# app/models/order.py - ORM Model (infrastructure)
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy import ForeignKey, String, Numeric

class OrderModel(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    status: Mapped[str] = mapped_column(String(50))
    total: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    items: Mapped[list["OrderItemModel"]] = relationship(back_populates="order")

    def to_domain(self) -> "Order":
        """Map ORM model to domain entity."""
        from app.domain.aggregates.order import Order, OrderItem
        from app.domain.value_objects import Money

        return Order(
            id=self.id,
            user_id=self.user_id,
            status=self.status,
            items=[
                OrderItem(
                    product_id=item.product_id,
                    product_name=item.product_name,
                    unit_price=Money(item.unit_price),
                    quantity=item.quantity,
                )
                for item in self.items
            ],
            created_at=self.created_at,
        )

    @classmethod
    def from_domain(cls, entity: "Order") -> "OrderModel":
        """Map domain entity to ORM model."""
        model = cls(
            id=entity.id,
            user_id=entity.user_id,
            status=entity.status,
            total=entity.total.amount,
        )
        model.items = [
            OrderItemModel(
                product_id=item.product_id,
                product_name=item.product_name,
                unit_price=item.unit_price.amount,
                quantity=item.quantity,
            )
            for item in entity.items
        ]
        return model
```

### Data Mapper in Repository

Encapsulate mapping logic within the repository to keep it out of the domain layer:

```python
# app/adapters/persistence/order_repository.py
class SqlAlchemyOrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def save(self, order: Order) -> Order:
        if order.id:
            model = await self.session.get(
                OrderModel, order.id,
                options=[selectinload(OrderModel.items)]
            )
            if not model:
                raise RepositoryError(f"Order {order.id} not found")
            self._sync_to_model(model, order)
        else:
            model = OrderModel.from_domain(order)
            self.session.add(model)

        await self.session.flush()
        await self.session.refresh(model, ["items"])
        return model.to_domain()

    def _sync_to_model(self, model: OrderModel, entity: Order) -> None:
        """Update existing model from domain entity, handling item changes."""
        model.status = entity.status
        model.total = entity.total.amount

        existing_ids = {item.product_id for item in model.items}
        entity_ids = {item.product_id for item in entity.items}

        # Remove items no longer in the entity
        model.items = [i for i in model.items if i.product_id in entity_ids]

        # Add new items
        for item in entity.items:
            if item.product_id not in existing_ids:
                model.items.append(OrderItemModel(
                    product_id=item.product_id,
                    product_name=item.product_name,
                    unit_price=item.unit_price.amount,
                    quantity=item.quantity,
                ))

        # Update existing items
        for model_item in model.items:
            entity_item = next(
                (i for i in entity.items if i.product_id == model_item.product_id), None
            )
            if entity_item:
                model_item.quantity = entity_item.quantity
                model_item.unit_price = entity_item.unit_price.amount
```

### When to Skip Domain Entities

Not every module needs a rich domain model. For simple CRUD modules with minimal business rules, mapping directly from Pydantic schemas to ORM models is appropriate:

```python
# Simple CRUD module - domain entity adds no value
# app/modules/categories/service.py
class CategoryService:
    def __init__(self, repo: CategoryRepository = Depends()):
        self.repo = repo

    async def create(self, data: CategoryCreate) -> Category:
        return await self.repo.create(**data.model_dump())

    async def update(self, id: int, data: CategoryUpdate) -> Category:
        updates = data.model_dump(exclude_unset=True)
        return await self.repo.update_by_id(id, **updates)
```

Use rich domain entities only when the domain has significant business rules, invariants to protect, or complex state transitions.

---

## Domain Error Hierarchy

```python
# app/domain/errors.py

class DomainError(Exception):
    """Base class for all domain-level errors."""
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class InsufficientStockError(DomainError):
    def __init__(self, product_name: str, requested: int = 0, available: int = 0):
        msg = f"Insufficient stock for '{product_name}'"
        if requested and available:
            msg += f": requested {requested}, available {available}"
        super().__init__(msg)
        self.product_name = product_name
        self.requested = requested
        self.available = available

class InvalidStateTransitionError(DomainError):
    def __init__(self, entity: str, current_status: str, target_status: str):
        super().__init__(
            f"{entity} cannot transition from '{current_status}' to '{target_status}'"
        )

class BusinessRuleViolation(DomainError):
    def __init__(self, rule: str, detail: str = ""):
        msg = f"Business rule violated: {rule}"
        if detail:
            msg += f" - {detail}"
        super().__init__(msg)
```

Map domain errors to HTTP responses in a centralized exception handler:

```python
# app/shared/exceptions.py
from app.domain.errors import DomainError, InsufficientStockError

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    status_map = {
        InsufficientStockError: 409,
        InvalidStateTransitionError: 409,
        BusinessRuleViolation: 422,
    }
    status_code = status_map.get(type(exc), 400)
    return JSONResponse(
        status_code=status_code,
        content={"error": type(exc).__name__, "detail": exc.message},
    )
```
