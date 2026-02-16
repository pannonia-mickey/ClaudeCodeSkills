---
name: Database Performance
description: This skill should be used when the user asks about "slow query", "EXPLAIN ANALYZE", "query optimization", "database index strategy", "connection pool tuning", "read replicas", "database performance", "N+1 query", or "database scaling". It covers query optimization, indexing strategies, execution plan analysis, and scaling patterns.
---

# Database Performance

## EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2025-01-01'
GROUP BY u.name
ORDER BY order_count DESC
LIMIT 10;

-- Key things to look for:
-- Seq Scan → Consider adding an index
-- Nested Loop with high rows → May need a different join strategy
-- Sort → Consider index for ORDER BY
-- actual time vs estimated → Statistics may be stale (run ANALYZE)
-- Buffers: shared hit vs read → Cache efficiency
```

## Index Strategy

```sql
-- Rule: Index columns in WHERE, JOIN, ORDER BY
-- Order composite indexes: equality → range → sort

-- Query: WHERE status = 'active' AND created_at > '2025-01-01' ORDER BY name
CREATE INDEX idx_users_status_created_name ON users(status, created_at, name);
--                                            equality  range       sort

-- Covering index (avoids table lookup)
CREATE INDEX idx_orders_user_cover ON orders(user_id) INCLUDE (status, total);

-- Partial index (smaller, faster)
CREATE INDEX idx_pending_orders ON orders(created_at) WHERE status = 'pending';

-- Expression index
CREATE INDEX idx_users_lower_email ON users(lower(email));
```

## Common Performance Issues

### N+1 Query

```sql
-- BAD: 1 query for users + N queries for orders
SELECT * FROM users;
-- For each user: SELECT * FROM orders WHERE user_id = ?

-- GOOD: Single query with JOIN
SELECT u.*, o.id AS order_id, o.total
FROM users u LEFT JOIN orders o ON o.user_id = u.id;

-- GOOD: Batch with IN clause
SELECT * FROM orders WHERE user_id IN ($1, $2, $3, ...);
```

### Missing Pagination

```sql
-- BAD: Fetch all rows
SELECT * FROM products;

-- GOOD: Paginate
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 0;
```

## Connection Pool Tuning

```
# PostgreSQL: max_connections = 100 (default)
# Each connection uses ~10MB RAM
# Pool size = (core_count * 2) + effective_spindle_count

# PgBouncer recommended settings:
pool_mode = transaction    # Best for web apps
max_client_conn = 1000     # Accept up to 1000 app connections
default_pool_size = 20     # Only 20 actual PG connections
reserve_pool_size = 5      # Extra connections for burst
```

## Read Replicas

```sql
-- Application-level read/write splitting
-- Write queries → Primary
-- Read queries → Replica(s)

-- Check replication lag
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- Ensure read-after-write consistency for critical paths:
-- After INSERT/UPDATE, read from primary for that user's session
```

## References

- [Optimization Techniques](references/optimization-techniques.md) — Query rewriting, materialized views, table partitioning, VACUUM tuning.
- [Scaling Patterns](references/scaling-patterns.md) — Sharding, read replicas, connection pooling, caching layers.
