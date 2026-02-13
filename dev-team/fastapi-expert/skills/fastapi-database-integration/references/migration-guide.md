# Alembic Migration Guide for FastAPI

This reference covers Alembic configuration for SQLModel and async SQLAlchemy, migration generation, data migrations, multi-database setups, and CI/CD integration patterns for FastAPI applications.

---

## Alembic Async Setup

### Installation and Initialization

```bash
pip install alembic
alembic init -t async alembic
```

The `-t async` flag generates an `env.py` template configured for async engines. This is essential when using `asyncpg` or `aiosqlite`.

### Project Structure

```
project/
    alembic/
        versions/           # Migration files
        env.py              # Migration environment configuration
        script.py.mako      # Migration file template
    alembic.ini             # Alembic configuration
    app/
        models/             # SQLModel table models
            __init__.py     # Import all models here for autogenerate
            user.py
            order.py
            product.py
```

### alembic.ini Configuration

```ini
[alembic]
script_location = alembic
prepend_sys_path = .

# Use async driver
sqlalchemy.url = postgresql+asyncpg://user:password@localhost:5432/mydb

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

### env.py for Async Migrations with SQLModel

The key difference from plain SQLAlchemy: use `SQLModel.metadata` as the target metadata. All SQLModel `table=True` classes automatically register their tables here.

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config

from sqlmodel import SQLModel
# Import all table models so they register with SQLModel.metadata
from app.models import user, order, product  # noqa: F401
from app.config import get_settings

config = context.config
settings = get_settings()

# Override URL from settings (environment-based)
config.set_main_option("sqlalchemy.url", settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = SQLModel.metadata


def run_migrations_offline() -> None:
    """Run migrations in offline mode (generates SQL script)."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
        compare_server_default=True,
    )

    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection) -> None:
    """Configure and run migrations with the given connection."""
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,
        compare_server_default=True,
        include_schemas=True,
        render_as_batch=True,  # Required for SQLite
    )

    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Run migrations in online mode with async engine."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()


def run_migrations_online() -> None:
    """Entry point for online migrations."""
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Mixing SQLModel and Plain SQLAlchemy Models

If you have some models defined with SQLModel (`table=True`) and others with plain SQLAlchemy `DeclarativeBase` (e.g., for optimistic locking or advanced mapper args), you need Alembic to see both sets of tables.

Option A — Use `SQLModel.metadata` everywhere by setting it on your `DeclarativeBase`:

```python
from sqlalchemy.orm import DeclarativeBase
from sqlmodel import SQLModel


class Base(DeclarativeBase):
    metadata = SQLModel.metadata  # Share the same metadata registry


class Product(Base):
    __tablename__ = "products"
    # ... Mapped columns, __mapper_args__, etc.
```

With this approach, `target_metadata = SQLModel.metadata` in `env.py` captures all tables.

Option B — Merge multiple metadata objects in `env.py`:

```python
from sqlmodel import SQLModel
from app.models.base import Base  # Your plain SQLAlchemy DeclarativeBase

from sqlalchemy import MetaData

target_metadata = MetaData()

# Reflect both metadata registries into a single target
for table in SQLModel.metadata.tables.values():
    table.tometadata(target_metadata)
for table in Base.metadata.tables.values():
    table.tometadata(target_metadata)
```

Option A is simpler and recommended.

---

## Auto-Generation of Migrations

### Generating from Model Changes

After modifying SQLModel table models, auto-generate a migration:

```bash
alembic revision --autogenerate -m "add_users_table"
```

Alembic compares `SQLModel.metadata` against the database schema and generates `upgrade()` and `downgrade()` functions.

### What Autogenerate Detects

Detected automatically:
- Table additions and removals
- Column additions, removals, and type changes
- Index and unique constraint changes
- Foreign key changes
- Nullable changes

Not detected automatically (must be written manually):
- Table or column renames
- Changes to column data
- Custom check constraints
- Enum value changes
- Trigger or function changes

### Reviewing Generated Migrations

Always review auto-generated migrations before applying them. A typical migration file:

```python
"""add_users_table

Revision ID: a1b2c3d4e5f6
Revises:
Create Date: 2025-01-15 10:30:00.000000
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa
import sqlmodel  # Required if migrations reference sqlmodel.sql.sqltypes.AutoString

revision: str = "a1b2c3d4e5f6"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    op.create_table(
        "users",
        sa.Column("email", sqlmodel.sql.sqltypes.AutoString(length=255), nullable=False),
        sa.Column("full_name", sqlmodel.sql.sqltypes.AutoString(length=100), nullable=False),
        sa.Column("is_active", sa.Boolean(), nullable=False),
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("hashed_password", sqlmodel.sql.sqltypes.AutoString(length=255), nullable=False),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("email"),
    )
    op.create_index("ix_users_email", "users", ["email"])


def downgrade() -> None:
    op.drop_index("ix_users_email", table_name="users")
    op.drop_table("users")
```

Note: SQLModel uses `sqlmodel.sql.sqltypes.AutoString` instead of `sa.String` for string columns. Auto-generated migrations will include `import sqlmodel` when these types are present. This is normal and expected.

---

## Data Migrations

Data migrations modify existing data rather than schema structure. These require special care.

### Schema + Data Migration

```python
"""add_user_role_column_with_default

Revision ID: b2c3d4e5f6g7
Revises: a1b2c3d4e5f6
"""
from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    # 1. Add column as nullable first
    op.add_column("users", sa.Column("role", sa.String(50), nullable=True))

    # 2. Backfill existing rows
    op.execute("UPDATE users SET role = 'member' WHERE role IS NULL")

    # 3. Make column non-nullable
    op.alter_column("users", "role", nullable=False, server_default="member")

def downgrade() -> None:
    op.drop_column("users", "role")
```

### Pure Data Migration

```python
"""normalize_email_addresses

Revision ID: c3d4e5f6g7h8
Revises: b2c3d4e5f6g7
"""
from alembic import op

def upgrade() -> None:
    # Normalize all email addresses to lowercase
    op.execute("UPDATE users SET email = LOWER(email)")

def downgrade() -> None:
    # Data migrations are often irreversible
    pass
```

### Batch Data Migration for Large Tables

For tables with millions of rows, process in batches to avoid long locks:

```python
"""backfill_user_slugs

Revision ID: d4e5f6g7h8i9
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy import text

def upgrade() -> None:
    conn = op.get_bind()

    # Add column
    op.add_column("users", sa.Column("slug", sa.String(255), nullable=True))

    # Process in batches
    batch_size = 5000
    while True:
        result = conn.execute(
            text("""
                UPDATE users
                SET slug = LOWER(REPLACE(full_name, ' ', '-'))
                WHERE id IN (
                    SELECT id FROM users
                    WHERE slug IS NULL
                    LIMIT :batch_size
                )
            """),
            {"batch_size": batch_size},
        )
        if result.rowcount == 0:
            break

    # Add unique index and make non-nullable
    op.create_index("ix_users_slug", "users", ["slug"], unique=True)
    op.alter_column("users", "slug", nullable=False)

def downgrade() -> None:
    op.drop_index("ix_users_slug", table_name="users")
    op.drop_column("users", "slug")
```

---

## Multi-Database Support

### Separate Alembic Environments

For applications using multiple databases (e.g., primary + analytics), maintain separate Alembic configurations:

```
alembic/
    primary/
        env.py
        versions/
    analytics/
        env.py
        versions/
alembic_primary.ini
alembic_analytics.ini
```

Run migrations for each database independently:

```bash
alembic -c alembic_primary.ini upgrade head
alembic -c alembic_analytics.ini upgrade head
```

### Schema-Based Multi-Tenancy

For schema-per-tenant architectures:

```python
"""create_tenant_schema

Revision ID: e5f6g7h8i9j0
"""
from alembic import op

def upgrade() -> None:
    # Template migration - called per tenant
    schema = context.get_x_argument(as_dictionary=True).get("schema", "public")

    op.execute(f"CREATE SCHEMA IF NOT EXISTS {schema}")

    with op.batch_alter_table("users", schema=schema) as batch_op:
        batch_op.create_table(
            sa.Column("id", sa.Integer(), primary_key=True),
            sa.Column("email", sa.String(255), nullable=False),
        )
```

Run per-tenant:

```bash
alembic upgrade head -x schema=tenant_acme
alembic upgrade head -x schema=tenant_globex
```

---

## CI/CD Integration

### Migration Validation in CI

Add a CI step that verifies migrations are consistent and complete:

```yaml
# .github/workflows/migrations.yml
name: Migration Check
on: [pull_request]

jobs:
  check-migrations:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run migrations
        env:
          APP_DATABASE_URL: postgresql+asyncpg://test_user:test_pass@localhost:5432/test_db
        run: alembic upgrade head

      - name: Verify no pending migrations
        env:
          APP_DATABASE_URL: postgresql+asyncpg://test_user:test_pass@localhost:5432/test_db
        run: |
          # Generate a migration - if anything is generated, models are out of sync
          alembic revision --autogenerate -m "check" 2>&1 | tee output.txt
          if grep -q "Detected" output.txt; then
            echo "ERROR: Pending model changes not captured in migrations"
            exit 1
          fi

      - name: Test downgrade
        env:
          APP_DATABASE_URL: postgresql+asyncpg://test_user:test_pass@localhost:5432/test_db
        run: |
          alembic downgrade -1
          alembic upgrade head
```

### Zero-Downtime Migration Strategy

For production deployments, follow this sequence:

1. **Expand**: Add new columns as nullable, add new tables, add new indexes concurrently.
2. **Migrate**: Deploy application code that writes to both old and new structures.
3. **Contract**: Remove old columns, old tables, and old indexes.

```python
# Step 1: Expand (migration)
def upgrade() -> None:
    op.add_column("users", sa.Column("username", sa.String(100), nullable=True))
    # Create index concurrently to avoid table locks (PostgreSQL)
    op.execute(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_users_username ON users (username)"
    )

# Step 2: Backfill (separate migration or background job)
def upgrade() -> None:
    op.execute(
        "UPDATE users SET username = LOWER(REPLACE(full_name, ' ', '_')) WHERE username IS NULL"
    )

# Step 3: Contract (later migration, after app no longer uses old pattern)
def upgrade() -> None:
    op.alter_column("users", "username", nullable=False)
```

### Deployment Script

```bash
#!/bin/bash
set -e

echo "Running database migrations..."
alembic upgrade head

echo "Starting application..."
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

For Docker:

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Run migrations and start app
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4"]
```

---

## Common Migration Commands

```bash
# Generate migration from model changes
alembic revision --autogenerate -m "description"

# Create empty migration for manual edits
alembic revision -m "description"

# Apply all pending migrations
alembic upgrade head

# Apply next N migrations
alembic upgrade +1

# Rollback last migration
alembic downgrade -1

# Rollback to specific revision
alembic downgrade a1b2c3d4

# Show current revision
alembic current

# Show migration history
alembic history --verbose

# Show pending migrations
alembic heads

# Generate SQL script instead of applying
alembic upgrade head --sql > migration.sql

# Stamp database at a revision without running migrations
alembic stamp head
```

---

## Troubleshooting

### "Target database is not up to date"

This occurs when the database has migrations not in the current branch. Resolve by checking which revision the database is at:

```bash
alembic current
alembic history
```

### Duplicate Head Revisions

When two branches create migrations simultaneously:

```bash
# Show all heads
alembic heads

# Merge divergent heads
alembic merge -m "merge_heads" rev1 rev2
```

### Circular Dependencies Between Tables

Use `use_alter=True` on foreign keys that create circular references. With SQLModel:

```python
from sqlmodel import Field, SQLModel

class Order(SQLModel, table=True):
    __tablename__ = "orders"

    id: int | None = Field(default=None, primary_key=True)
    last_item_id: int | None = Field(
        default=None,
        sa_column_kwargs={
            "use_alter": True,
            "name": "fk_order_last_item",
        },
        foreign_key="order_items.id",
    )
```

For more complex cases, define the foreign key constraint separately using `__table_args__`:

```python
from sqlalchemy import ForeignKeyConstraint
from sqlmodel import Field, SQLModel

class Order(SQLModel, table=True):
    __tablename__ = "orders"
    __table_args__ = (
        ForeignKeyConstraint(
            ["last_item_id"], ["order_items.id"],
            name="fk_order_last_item",
            use_alter=True,
        ),
    )

    id: int | None = Field(default=None, primary_key=True)
    last_item_id: int | None = Field(default=None)
```

### SQLModel AutoString in Migrations

When auto-generating migrations, SQLModel string columns produce `sqlmodel.sql.sqltypes.AutoString` instead of `sa.String`. This is expected. Ensure `sqlmodel` is importable in the migration environment. If you prefer standard SQLAlchemy types in migration files, you can customize the `render_item` function in `env.py`:

```python
def render_item(type_, obj, autogen_context):
    if type_ == "type" and isinstance(obj, sqlmodel.sql.sqltypes.AutoString):
        autogen_context.imports.add("import sqlalchemy as sa")
        if obj.length:
            return f"sa.String(length={obj.length})"
        return "sa.String()"
    return False

# In do_run_migrations:
context.configure(
    connection=connection,
    target_metadata=target_metadata,
    render_item=render_item,
    # ...
)
```
