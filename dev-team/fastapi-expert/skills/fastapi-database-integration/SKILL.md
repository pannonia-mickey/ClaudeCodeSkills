---
name: FastAPI Database Integration
description: This skill should be used when the user asks about "FastAPI database", "SQLModel", "SQLAlchemy FastAPI", "Alembic", "FastAPI repository", or "async database". It covers SQLModel model hierarchies, SQLAlchemy 2.0 async patterns, session management, the repository pattern, Alembic migrations, connection pooling, and transaction handling. Use this skill for any task involving database models, query optimization, migration management, or data access layer design in a FastAPI context.
---

### Choosing Between SQLModel and SQLAlchemy

SQLModel (by the creator of FastAPI) merges SQLAlchemy and Pydantic into a single class, eliminating the need to maintain separate ORM models and API schemas. Use SQLModel as the default for FastAPI projects. Fall back to plain SQLAlchemy `Mapped`/`mapped_column` only when you need features SQLModel does not yet expose (e.g., composite indexes via `__table_args__`, advanced mapper configurations, or column-level `server_default` with SQL expressions).

| Concern | SQLModel | Plain SQLAlchemy 2.0 |
|---|---|---|
| Model + Schema in one class | Yes (`table=True`) | No (separate Pydantic models) |
| Pydantic validation built-in | Yes | Manual |
| Relationships | `Relationship()` | `relationship()` + `mapped_column` |
| Async support | Via SQLAlchemy AsyncSession | Native |
| Alembic integration | `SQLModel.metadata` | `Base.metadata` |

### SQLModel Model Hierarchy

Define a base schema, a table model, and purpose-specific request/response schemas. This prevents leaking internal fields (like `hashed_password`) to API consumers and keeps input validation separate from storage concerns.

```python
from sqlmodel import Field, Relationship, SQLModel


# -- Shared fields (never a table) --
class UserBase(SQLModel):
    email: str = Field(max_length=255, index=True, unique=True)
    full_name: str = Field(max_length=100)
    is_active: bool = Field(default=True)


# -- Database table --
class User(UserBase, table=True):
    __tablename__ = "users"

    id: int | None = Field(default=None, primary_key=True)
    hashed_password: str = Field(max_length=255)

    orders: list["Order"] = Relationship(back_populates="user")


# -- API input (create) --
class UserCreate(UserBase):
    password: str = Field(min_length=8)


# -- API output --
class UserPublic(UserBase):
    id: int


# -- API input (partial update) --
class UserUpdate(SQLModel):
    email: str | None = None
    full_name: str | None = None
    is_active: bool | None = None
    password: str | None = Field(default=None, min_length=8)
```

For related models, use `Field(foreign_key=...)` and `Relationship()`:

```python
from datetime import datetime
from decimal import Decimal

from sqlmodel import Field, Relationship, SQLModel


class OrderBase(SQLModel):
    total: Decimal = Field(decimal_places=2)
    status: str = Field(default="pending", max_length=50)


class Order(OrderBase, table=True):
    __tablename__ = "orders"

    id: int | None = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="users.id", index=True)
    created_at: datetime | None = Field(default=None)
    updated_at: datetime | None = Field(default=None)

    user: "User" = Relationship(back_populates="orders")
    items: list["OrderItem"] = Relationship(back_populates="order")


class OrderCreate(OrderBase):
    user_id: int


class OrderPublic(OrderBase):
    id: int
    user_id: int
    created_at: datetime | None = None


class OrderItem(SQLModel, table=True):
    __tablename__ = "order_items"

    id: int | None = Field(default=None, primary_key=True)
    order_id: int = Field(foreign_key="orders.id", index=True)
    product_id: int = Field(foreign_key="products.id", index=True)
    quantity: int = Field(gt=0)
    price: Decimal = Field(decimal_places=2)

    order: "Order" = Relationship(back_populates="items")
```

### Engine and Session Setup (Async)

Configure an async engine and session factory for production use with connection pooling:

```python
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost:5432/mydb",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600,
    echo=False,
)

async_session_factory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

Provide sessions as a FastAPI dependency with automatic commit/rollback:

```python
from collections.abc import AsyncGenerator
from fastapi import Depends

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

Initialize and dispose the engine within the application lifespan:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

### Engine and Session Setup (Sync)

For simpler applications or prototyping, SQLModel provides a synchronous `Session`:

```python
from sqlmodel import Session, create_engine

engine = create_engine(
    "postgresql://user:pass@localhost:5432/mydb",
    echo=False,
)

def get_session():
    with Session(engine) as session:
        yield session
```

### CRUD Operations with SQLModel

SQLModel's `model_validate()` converts between schema types, and `sqlmodel_update()` applies partial updates:

```python
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select

router = APIRouter(prefix="/users", tags=["users"])


@router.post("/", response_model=UserPublic, status_code=201)
async def create_user(
    user_in: UserCreate,
    session: AsyncSession = Depends(get_db),
):
    db_user = User.model_validate(
        user_in, update={"hashed_password": hash_password(user_in.password)}
    )
    session.add(db_user)
    await session.flush()
    await session.refresh(db_user)
    return db_user


@router.get("/", response_model=list[UserPublic])
async def list_users(
    session: AsyncSession = Depends(get_db),
    offset: int = 0,
    limit: int = Query(default=20, le=100),
):
    result = await session.execute(
        select(User).offset(offset).limit(limit)
    )
    return result.scalars().all()


@router.get("/{user_id}", response_model=UserPublic)
async def get_user(
    user_id: int,
    session: AsyncSession = Depends(get_db),
):
    user = await session.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user


@router.patch("/{user_id}", response_model=UserPublic)
async def update_user(
    user_id: int,
    user_in: UserUpdate,
    session: AsyncSession = Depends(get_db),
):
    db_user = await session.get(User, user_id)
    if not db_user:
        raise HTTPException(status_code=404, detail="User not found")
    update_data = user_in.model_dump(exclude_unset=True)
    if "password" in update_data:
        update_data["hashed_password"] = hash_password(update_data.pop("password"))
    db_user.sqlmodel_update(update_data)
    session.add(db_user)
    await session.flush()
    await session.refresh(db_user)
    return db_user


@router.delete("/{user_id}", status_code=204)
async def delete_user(
    user_id: int,
    session: AsyncSession = Depends(get_db),
):
    user = await session.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    await session.delete(user)
    await session.flush()
```

### Repository Pattern with SQLModel

Wrap data access behind a generic repository. SQLModel table models work as the `ModelType`:

```python
from typing import TypeVar, Generic, Type, Sequence, Any
from sqlalchemy import func
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import SQLModel, select

ModelType = TypeVar("ModelType", bound=SQLModel)


class BaseRepository(Generic[ModelType]):
    def __init__(self, model: Type[ModelType], session: AsyncSession):
        self.model = model
        self.session = session

    async def get_by_id(self, id: int) -> ModelType | None:
        return await self.session.get(self.model, id)

    async def get_many(
        self, skip: int = 0, limit: int = 20
    ) -> Sequence[ModelType]:
        result = await self.session.execute(
            select(self.model).offset(skip).limit(limit)
        )
        return result.scalars().all()

    async def count(self) -> int:
        result = await self.session.execute(
            select(func.count()).select_from(self.model)
        )
        return result.scalar_one()

    async def create_from_schema(self, schema: SQLModel, **overrides) -> ModelType:
        instance = self.model.model_validate(schema, update=overrides)
        self.session.add(instance)
        await self.session.flush()
        await self.session.refresh(instance)
        return instance

    async def update_instance(self, instance: ModelType, data: dict) -> ModelType:
        instance.sqlmodel_update(data)
        self.session.add(instance)
        await self.session.flush()
        await self.session.refresh(instance)
        return instance

    async def delete_instance(self, instance: ModelType) -> None:
        await self.session.delete(instance)
        await self.session.flush()
```

Create domain-specific repositories that extend the base:

```python
class UserRepository(BaseRepository[User]):
    def __init__(self, session: AsyncSession = Depends(get_db)):
        super().__init__(User, session)

    async def get_by_email(self, email: str) -> User | None:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def get_active_users(
        self, skip: int = 0, limit: int = 20
    ) -> Sequence[User]:
        result = await self.session.execute(
            select(User)
            .where(User.is_active == True)
            .offset(skip)
            .limit(limit)
        )
        return result.scalars().all()
```

### Session Management and Transactions

Handle transactions spanning multiple repositories with a Unit of Work:

```python
class UnitOfWork:
    def __init__(self, session: AsyncSession = Depends(get_db)):
        self.session = session

    async def __aenter__(self):
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            await self.session.rollback()
        else:
            await self.session.commit()

    @property
    def users(self) -> UserRepository:
        return UserRepository(self.session)

    @property
    def orders(self) -> OrderRepository:
        return OrderRepository(self.session)
```

Use nested transactions (savepoints) for partial rollback:

```python
async def transfer_credits(
    uow: UnitOfWork,
    from_user_id: int,
    to_user_id: int,
    amount: Decimal,
):
    async with uow.session.begin_nested():
        sender = await uow.users.get_by_id(from_user_id)
        receiver = await uow.users.get_by_id(to_user_id)

        if sender.credits < amount:
            raise InsufficientCreditsError()

        sender.credits -= amount
        receiver.credits += amount
        await uow.session.flush()
```

### Connection Pooling Configuration

Configure connection pooling for production workloads:

```python
engine = create_async_engine(
    settings.database_url,
    # Pool sizing
    pool_size=settings.db_pool_size,          # Steady-state connections
    max_overflow=settings.db_max_overflow,     # Burst capacity
    # Connection health
    pool_pre_ping=True,                        # Verify connections before use
    pool_recycle=1800,                         # Recycle connections after 30 min
    # Timeout
    pool_timeout=30,                           # Wait max 30s for a connection
    # Performance
    echo=settings.debug,                       # SQL logging in dev only
    echo_pool=settings.debug,                  # Pool event logging in dev only
)
```

Guidelines for pool sizing:
- Set `pool_size` to the expected number of concurrent requests per worker.
- Set `max_overflow` to handle traffic spikes without over-provisioning.
- With multiple Uvicorn workers, the total connections equal `pool_size * num_workers`. Ensure the database `max_connections` setting accommodates this.

### Eager Loading and N+1 Prevention

Avoid N+1 query problems with explicit eager loading:

```python
from sqlalchemy.orm import selectinload, joinedload
from sqlmodel import select

# selectinload: Loads related items via a separate IN query (good for collections)
stmt = (
    select(User)
    .where(User.id == user_id)
    .options(selectinload(User.orders).selectinload(Order.items))
)

# joinedload: Loads via JOIN (good for single related objects)
stmt = (
    select(Order)
    .where(Order.id == order_id)
    .options(joinedload(Order.user))
)
```

Create reusable query options:

```python
USER_WITH_ORDERS = [selectinload(User.orders)]
ORDER_WITH_DETAILS = [
    joinedload(Order.user),
    selectinload(Order.items),
]

async def get_user_with_orders(
    session: AsyncSession, user_id: int
) -> User | None:
    result = await session.execute(
        select(User)
        .where(User.id == user_id)
        .options(*USER_WITH_ORDERS)
    )
    return result.scalar_one_or_none()
```

### Falling Back to SQLAlchemy for Advanced Models

When you need features that SQLModel does not directly expose (composite indexes, server-side defaults with SQL expressions, mapper arguments for optimistic locking), define the model with plain SQLAlchemy and use SQLModel schemas alongside it:

```python
from datetime import datetime
from sqlalchemy import Index, String, func, text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "products"
    __table_args__ = (
        Index("ix_products_category_price", "category_id", "price"),
        Index(
            "ix_products_name_trgm", "name",
            postgresql_using="gin",
            postgresql_ops={"name": "gin_trgm_ops"},
        ),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    category_id: Mapped[int] = mapped_column(index=True)
    price: Mapped[float] = mapped_column()
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    version: Mapped[int] = mapped_column(default=1)

    __mapper_args__ = {"version_id_col": version}


# SQLModel schemas for API I/O
class ProductPublic(SQLModel):
    id: int
    name: str
    price: float
    is_active: bool
```

### Alembic Configuration with SQLModel

When using SQLModel, reference `SQLModel.metadata` instead of a custom `Base.metadata`:

```python
# alembic/env.py
from sqlmodel import SQLModel

# Import all table models so they register with SQLModel.metadata
from app.models import user, order, product  # noqa: F401

target_metadata = SQLModel.metadata
```

Auto-generate and apply migrations:

```bash
alembic revision --autogenerate -m "add_users_table"
alembic upgrade head
```

When mixing SQLModel tables with plain SQLAlchemy `DeclarativeBase` models, use a shared metadata or configure Alembic to inspect both. The simplest approach is to have all SQLModel table models imported before Alembic reads the metadata, and set plain SQLAlchemy models to also use `SQLModel.metadata` via `metadata=SQLModel.metadata` in `DeclarativeBase`.

## References

- [references/database-patterns.md](references/database-patterns.md) - Unit of Work, repository pattern, async sessions, transactions, query builder patterns, and advanced SQLAlchemy techniques with SQLModel
- [references/migration-guide.md](references/migration-guide.md) - Alembic async setup, auto-generation, data migrations, multi-database support, and CI/CD integration
