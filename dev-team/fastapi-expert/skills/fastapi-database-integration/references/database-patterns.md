# Database Patterns for FastAPI

This reference provides advanced SQLModel and SQLAlchemy 2.0 patterns for building robust, maintainable data access layers in FastAPI applications. SQLModel is the default choice for model definitions; plain SQLAlchemy is used where advanced ORM features are required. Each pattern includes complete async Python code ready for production use.

---

## Unit of Work Pattern

The Unit of Work pattern coordinates writes across multiple repositories within a single transaction. It ensures atomicity: either all changes commit or all roll back.

### Core Implementation

```python
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import Depends

class UnitOfWork:
    """
    Coordinates transactional work across multiple repositories.
    Usage: inject via Depends(), use as async context manager.
    """

    def __init__(self, session: AsyncSession = Depends(get_db)):
        self.session = session
        self._users: UserRepository | None = None
        self._orders: OrderRepository | None = None
        self._products: ProductRepository | None = None

    async def __aenter__(self) -> "UnitOfWork":
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        if exc_type is not None:
            await self.rollback()

    async def commit(self) -> None:
        await self.session.commit()

    async def rollback(self) -> None:
        await self.session.rollback()

    @property
    def users(self) -> "UserRepository":
        if self._users is None:
            self._users = UserRepository(self.session)
        return self._users

    @property
    def orders(self) -> "OrderRepository":
        if self._orders is None:
            self._orders = OrderRepository(self.session)
        return self._orders

    @property
    def products(self) -> "ProductRepository":
        if self._products is None:
            self._products = ProductRepository(self.session)
        return self._products
```

### Transactional Service Using Unit of Work

```python
from decimal import Decimal
from sqlmodel import SQLModel


class OrderItemCreate(SQLModel):
    product_id: int
    quantity: int


class OrderService:
    def __init__(self, uow: UnitOfWork = Depends()):
        self.uow = uow

    async def place_order(
        self, user_id: int, items: list[OrderItemCreate]
    ) -> Order:
        async with self.uow:
            # Verify user exists
            user = await self.uow.users.get_by_id(user_id)
            if not user:
                raise NotFoundError("User", user_id)

            # Check and reserve stock
            total = Decimal("0")
            order_items = []
            for item in items:
                product = await self.uow.products.get_by_id(item.product_id)
                if not product:
                    raise NotFoundError("Product", item.product_id)
                if product.stock < item.quantity:
                    raise InsufficientStockError(product.name)

                product.stock -= item.quantity
                total += product.price * item.quantity
                order_items.append(
                    OrderItem(
                        product_id=product.id,
                        quantity=item.quantity,
                        price=product.price,
                    )
                )

            # Create order using SQLModel's model_validate
            order = await self.uow.orders.create_from_schema(
                OrderCreate(user_id=user_id, total=total, status="pending")
            )
            for oi in order_items:
                oi.order_id = order.id
                self.uow.session.add(oi)

            await self.uow.commit()
            return order
```

---

## Repository Pattern

### Generic Base Repository with SQLModel

The repository is generic over any SQLModel table model. It uses `model_validate()` for creating instances from schemas and `sqlmodel_update()` for partial updates.

```python
from typing import TypeVar, Generic, Type, Any, Sequence
from sqlalchemy import select, func, update, delete
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.sql import Select
from sqlmodel import SQLModel

ModelType = TypeVar("ModelType", bound=SQLModel)


class BaseRepository(Generic[ModelType]):
    """Generic repository providing standard CRUD operations for SQLModel tables."""

    model: Type[ModelType]

    def __init__(self, session: AsyncSession):
        self.session = session

    def _base_query(self) -> Select:
        """Override in subclasses to add default filters (e.g., soft delete)."""
        return select(self.model)

    async def get_by_id(
        self, id: int, options: list | None = None
    ) -> ModelType | None:
        stmt = self._base_query().where(self.model.id == id)
        if options:
            stmt = stmt.options(*options)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_many(
        self,
        skip: int = 0,
        limit: int = 20,
        order_by: Any = None,
        options: list | None = None,
    ) -> Sequence[ModelType]:
        stmt = self._base_query().offset(skip).limit(limit)
        if order_by is not None:
            stmt = stmt.order_by(order_by)
        if options:
            stmt = stmt.options(*options)
        result = await self.session.execute(stmt)
        return result.scalars().all()

    async def count(self, filters: list | None = None) -> int:
        stmt = select(func.count()).select_from(self.model)
        if filters:
            stmt = stmt.where(*filters)
        result = await self.session.execute(stmt)
        return result.scalar_one()

    async def create_from_schema(
        self, schema: SQLModel, **overrides
    ) -> ModelType:
        """Create a table instance from a SQLModel input schema."""
        instance = self.model.model_validate(schema, update=overrides)
        self.session.add(instance)
        await self.session.flush()
        await self.session.refresh(instance)
        return instance

    async def create(self, **kwargs) -> ModelType:
        """Create a table instance from keyword arguments."""
        instance = self.model(**kwargs)
        self.session.add(instance)
        await self.session.flush()
        await self.session.refresh(instance)
        return instance

    async def bulk_create(self, items: list[dict]) -> list[ModelType]:
        instances = [self.model(**item) for item in items]
        self.session.add_all(instances)
        await self.session.flush()
        for instance in instances:
            await self.session.refresh(instance)
        return instances

    async def update_by_id(self, id: int, data: dict) -> ModelType | None:
        """Update a record using sqlmodel_update for safe partial updates."""
        instance = await self.get_by_id(id)
        if instance is None:
            return None
        instance.sqlmodel_update(data)
        self.session.add(instance)
        await self.session.flush()
        await self.session.refresh(instance)
        return instance

    async def bulk_update(self, filters: list, **kwargs) -> int:
        stmt = (
            update(self.model)
            .where(*filters)
            .values(**kwargs)
            .execution_options(synchronize_session="fetch")
        )
        result = await self.session.execute(stmt)
        return result.rowcount

    async def delete_by_id(self, id: int) -> bool:
        instance = await self.get_by_id(id)
        if instance is None:
            return False
        await self.session.delete(instance)
        await self.session.flush()
        return True

    async def exists(self, **kwargs) -> bool:
        stmt = select(func.count()).select_from(self.model)
        for key, value in kwargs.items():
            stmt = stmt.where(getattr(self.model, key) == value)
        result = await self.session.execute(stmt)
        return result.scalar_one() > 0
```

### Soft Delete Repository

For SQLModel table models that include a `deleted_at` column:

```python
from datetime import datetime
from sqlalchemy import func
from sqlmodel import Field, SQLModel


class SoftDeleteMixin(SQLModel):
    """Mixin adding a deleted_at field for soft deletes. Add to table models."""
    deleted_at: datetime | None = Field(default=None)


class SoftDeleteRepository(BaseRepository[ModelType]):
    """Repository that supports soft deletes via a deleted_at column."""

    def _base_query(self) -> Select:
        return select(self.model).where(self.model.deleted_at.is_(None))

    async def soft_delete(self, id: int) -> bool:
        instance = await self.get_by_id(id)
        if instance is None:
            return False
        instance.deleted_at = func.now()
        await self.session.flush()
        return True

    async def restore(self, id: int) -> ModelType | None:
        # Bypass the soft-delete filter
        stmt = select(self.model).where(self.model.id == id)
        result = await self.session.execute(stmt)
        instance = result.scalar_one_or_none()
        if instance is None:
            return None
        instance.deleted_at = None
        await self.session.flush()
        return instance
```

Usage with a SQLModel table:

```python
class Article(SoftDeleteMixin, table=True):
    __tablename__ = "articles"

    id: int | None = Field(default=None, primary_key=True)
    title: str = Field(max_length=200)
    body: str
    author_id: int = Field(foreign_key="users.id")


class ArticleRepository(SoftDeleteRepository[Article]):
    model = Article
```

---

## Async Session Patterns

### Scoped Session for Background Tasks

Background tasks run outside the request lifecycle, so they need their own session:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def standalone_session():
    """Create an independent session for background tasks."""
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

async def process_order_background(order_id: int):
    async with standalone_session() as session:
        order = await session.get(Order, order_id)
        # Process the order...
        order.status = "processed"
        await session.commit()
```

### Read Replica Routing

Route read queries to a replica and writes to the primary:

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

primary_engine = create_async_engine(settings.primary_database_url)
replica_engine = create_async_engine(settings.replica_database_url)

primary_session_factory = async_sessionmaker(primary_engine, expire_on_commit=False)
replica_session_factory = async_sessionmaker(replica_engine, expire_on_commit=False)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Primary database session for writes."""
    async with primary_session_factory() as session:
        yield session


async def get_read_db() -> AsyncGenerator[AsyncSession, None]:
    """Replica session for read-only queries."""
    async with replica_session_factory() as session:
        yield session


@router.get("/users/{user_id}", response_model=UserPublic)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_read_db),  # Uses replica
):
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user


@router.post("/users/", response_model=UserPublic, status_code=201)
async def create_user(
    user_in: UserCreate,
    db: AsyncSession = Depends(get_db),  # Uses primary
):
    db_user = User.model_validate(
        user_in, update={"hashed_password": hash_password(user_in.password)}
    )
    db.add(db_user)
    await db.flush()
    await db.refresh(db_user)
    return db_user
```

---

## Transaction Patterns

### Nested Transactions with Savepoints

```python
from sqlmodel import SQLModel, Field

async def complex_operation(session: AsyncSession):
    # Outer transaction
    user = User(
        email="user@example.com",
        full_name="Test User",
        hashed_password="hashed",
    )
    session.add(user)
    await session.flush()

    try:
        # Inner savepoint - can fail independently
        async with session.begin_nested():
            notification = Notification(
                user_id=user.id,
                message="Welcome!",
            )
            session.add(notification)
            await session.flush()
    except IntegrityError:
        # Savepoint rolled back, but user creation still intact
        logger.warning("Failed to create notification, continuing")

    await session.commit()  # User is committed even if notification failed
```

### Optimistic Locking

SQLModel does not expose `__mapper_args__` directly. Use a plain SQLAlchemy model with a `version` column and SQLModel schemas for API I/O:

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String
from sqlmodel import SQLModel


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    stock: Mapped[int] = mapped_column(default=0)
    version: Mapped[int] = mapped_column(default=1)

    __mapper_args__ = {"version_id_col": version}


class ProductPublic(SQLModel):
    id: int
    name: str
    stock: int


async def decrement_stock(
    session: AsyncSession, product_id: int, quantity: int
) -> Product:
    product = await session.get(Product, product_id)
    if product.stock < quantity:
        raise InsufficientStockError()

    product.stock -= quantity
    try:
        await session.flush()
    except StaleDataError:
        await session.rollback()
        raise ConflictError(
            "Product", "version",
            "Product was modified by another request. Retry."
        )
    return product
```

---

## Query Builder Patterns

### Filterable Query Builder

Works with both SQLModel and SQLAlchemy models:

```python
from dataclasses import dataclass
from typing import Any, Generic, Type, TypeVar, Sequence
from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.sql import Select
from sqlmodel import SQLModel

ModelType = TypeVar("ModelType", bound=SQLModel)


@dataclass
class QueryFilter:
    field: str
    operator: str  # eq, ne, gt, gte, lt, lte, like, in, is_null
    value: Any


class QueryBuilder(Generic[ModelType]):
    def __init__(self, model: Type[ModelType]):
        self.model = model
        self._stmt = select(model)

    def filter(self, filters: list[QueryFilter]) -> "QueryBuilder":
        for f in filters:
            column = getattr(self.model, f.field)
            match f.operator:
                case "eq":
                    self._stmt = self._stmt.where(column == f.value)
                case "ne":
                    self._stmt = self._stmt.where(column != f.value)
                case "gt":
                    self._stmt = self._stmt.where(column > f.value)
                case "gte":
                    self._stmt = self._stmt.where(column >= f.value)
                case "lt":
                    self._stmt = self._stmt.where(column < f.value)
                case "lte":
                    self._stmt = self._stmt.where(column <= f.value)
                case "like":
                    self._stmt = self._stmt.where(column.ilike(f"%{f.value}%"))
                case "in":
                    self._stmt = self._stmt.where(column.in_(f.value))
                case "is_null":
                    if f.value:
                        self._stmt = self._stmt.where(column.is_(None))
                    else:
                        self._stmt = self._stmt.where(column.is_not(None))
        return self

    def order_by(self, field: str, desc: bool = False) -> "QueryBuilder":
        column = getattr(self.model, field)
        self._stmt = self._stmt.order_by(column.desc() if desc else column.asc())
        return self

    def paginate(self, skip: int = 0, limit: int = 20) -> "QueryBuilder":
        self._stmt = self._stmt.offset(skip).limit(limit)
        return self

    def with_options(self, *options) -> "QueryBuilder":
        self._stmt = self._stmt.options(*options)
        return self

    def build(self) -> Select:
        return self._stmt

    async def execute(self, session: AsyncSession) -> Sequence[ModelType]:
        result = await session.execute(self._stmt)
        return result.scalars().all()

    async def execute_count(self, session: AsyncSession) -> int:
        count_stmt = select(func.count()).select_from(self._stmt.subquery())
        result = await session.execute(count_stmt)
        return result.scalar_one()
```

### Usage in Repository

```python
class ProductBase(SQLModel):
    name: str = Field(max_length=200, index=True)
    category_id: int = Field(foreign_key="categories.id", index=True)
    price: Decimal = Field(decimal_places=2)
    is_active: bool = Field(default=True)


class Product(ProductBase, table=True):
    __tablename__ = "products"

    id: int | None = Field(default=None, primary_key=True)
    created_at: datetime | None = Field(default=None)


class ProductRepository(BaseRepository[Product]):
    model = Product

    async def search(
        self,
        filters: list[QueryFilter],
        sort_by: str = "created_at",
        sort_desc: bool = True,
        skip: int = 0,
        limit: int = 20,
    ) -> tuple[Sequence[Product], int]:
        builder = QueryBuilder(Product)
        builder.filter(filters).order_by(sort_by, desc=sort_desc)

        total = await builder.execute_count(self.session)
        items = await builder.paginate(skip, limit).execute(self.session)

        return items, total
```

---

## Streaming Large Result Sets

For queries returning thousands of rows, use server-side cursors to avoid loading everything into memory:

```python
from sqlmodel import select

async def stream_all_users(
    session: AsyncSession, batch_size: int = 1000
):
    """Yield users in batches using server-side cursor."""
    stmt = select(User).order_by(User.id)
    result = await session.stream(stmt)

    async for partition in result.partitions(batch_size):
        yield [row[0] for row in partition]
```

Usage in an endpoint:

```python
from fastapi.responses import StreamingResponse

@router.get("/exports/users")
async def export_users(db: AsyncSession = Depends(get_db)):
    async def generate():
        async for batch in stream_all_users(db):
            for user in batch:
                yield f"{user.id},{user.email},{user.full_name}\n"

    return StreamingResponse(generate(), media_type="text/csv")
```

---

## Index Strategy

SQLModel's `Field()` supports `index=True` and `unique=True` for simple indexes. For composite indexes, partial indexes, or database-specific index types, use `__table_args__` on the table model:

```python
from sqlalchemy import Index, text
from sqlmodel import Field, SQLModel


class Product(SQLModel, table=True):
    __tablename__ = "products"
    __table_args__ = (
        Index("ix_products_category_price", "category_id", "price"),
        Index(
            "ix_products_name_trgm", "name",
            postgresql_using="gin",
            postgresql_ops={"name": "gin_trgm_ops"},
        ),
        Index(
            "ix_products_active", "is_active",
            postgresql_where=text("is_active = true"),
        ),
    )

    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(max_length=200)
    category_id: int = Field(foreign_key="categories.id", index=True)
    price: Decimal = Field(decimal_places=2)
    is_active: bool = Field(default=True)
```

Note: `__table_args__` works on SQLModel `table=True` classes exactly as it does on plain SQLAlchemy `DeclarativeBase` models.
