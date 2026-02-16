---
name: Database Mastery
description: This skill should be used when the user asks about "database design", "normalization", "ACID properties", "CAP theorem", "database indexes", "transactions", "isolation levels", "connection pooling", or "database fundamentals". It covers core database concepts, schema design principles, and operational fundamentals.
---

# Database Fundamentals

## ACID Properties

- **Atomicity**: Transaction is all-or-nothing. If any part fails, the entire transaction rolls back.
- **Consistency**: Transaction brings the database from one valid state to another. Constraints are enforced.
- **Isolation**: Concurrent transactions don't interfere with each other.
- **Durability**: Once committed, data survives system crashes.

## Normalization

```sql
-- 1NF: Atomic values, no repeating groups
-- BAD: tags VARCHAR = 'js,react,node'
-- GOOD: Separate tags table with FK

-- 2NF: No partial dependencies (all non-key columns depend on entire PK)
-- 3NF: No transitive dependencies (non-key depends only on PK)

-- Example: 3NF design
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(254) NOT NULL UNIQUE,
  name VARCHAR(100) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  status VARCHAR(20) NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
  total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  quantity INTEGER NOT NULL CHECK (quantity > 0),
  unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0)
);
```

## Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|-------------------|-------------|
| Read Uncommitted | Yes | Yes | Yes |
| Read Committed | No | Yes | Yes |
| Repeatable Read | No | No | Yes (not in PG) |
| Serializable | No | No | No |

```sql
-- PostgreSQL default is Read Committed
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
  -- Critical section
COMMIT;
```

## Index Types

```sql
-- B-tree (default) — equality and range queries
CREATE INDEX idx_users_email ON users(email);

-- Composite — multi-column queries
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Partial — index subset of rows
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Covering — includes extra columns to avoid table lookup
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (status, total_amount);

-- GIN — arrays, JSONB, full-text search
CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- GiST — geometric, range, full-text
CREATE INDEX idx_locations_point ON locations USING GIST(coordinates);

-- BRIN — large tables with naturally ordered data
CREATE INDEX idx_events_created ON events USING BRIN(created_at);
```

## Connection Pooling

```typescript
// PgBouncer configuration (recommended for production)
// pgbouncer.ini
// pool_mode = transaction
// max_client_conn = 1000
// default_pool_size = 20

// Application-level pooling (Prisma)
const prisma = new PrismaClient({
  datasources: { db: { url: `${DATABASE_URL}?connection_limit=10&pool_timeout=10` } },
});
```

## References

- [Schema Patterns](references/schema-patterns.md) — Multi-tenant, audit trails, soft deletes, temporal tables, polymorphic associations.
- [Migration Strategies](references/migration-strategies.md) — Zero-downtime migrations, backfills, schema versioning, rollback patterns.
