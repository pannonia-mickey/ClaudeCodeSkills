---
name: SQL Design
description: This skill should be used when the user asks about "write SQL query", "window functions", "CTEs", "recursive query", "PostgreSQL JSONB", "stored procedures", "SQL triggers", "partitioning", "full-text search SQL", or "advanced SQL". It covers advanced SQL features for PostgreSQL, MySQL, and SQL Server.
---

# Advanced SQL Design

## Window Functions

```sql
-- Running total
SELECT date, revenue,
  SUM(revenue) OVER (ORDER BY date) AS running_total
FROM daily_sales;

-- Rank within groups
SELECT department, name, salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;

-- Moving average
SELECT date, value,
  AVG(value) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7d
FROM metrics;

-- Lead/Lag — compare with adjacent rows
SELECT month, revenue,
  revenue - LAG(revenue) OVER (ORDER BY month) AS month_over_month_change,
  ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month)) / LAG(revenue) OVER (ORDER BY month), 1) AS pct_change
FROM monthly_revenue;
```

## Common Table Expressions (CTEs)

```sql
-- Recursive CTE — organizational hierarchy
WITH RECURSIVE org_tree AS (
  SELECT id, name, manager_id, 0 AS depth
  FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id, ot.depth + 1
  FROM employees e JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT * FROM org_tree ORDER BY depth, name;

-- Materialized CTE for performance
WITH active_users AS MATERIALIZED (
  SELECT id, name FROM users WHERE active = true AND last_login > now() - INTERVAL '30 days'
)
SELECT au.name, COUNT(o.id) AS order_count
FROM active_users au JOIN orders o ON o.user_id = au.id
GROUP BY au.name;
```

## PostgreSQL JSONB

```sql
-- Store and query JSON
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(50) NOT NULL,
  payload JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Query JSONB
SELECT * FROM events WHERE payload->>'userId' = '123';
SELECT * FROM events WHERE payload @> '{"action": "click"}';
SELECT * FROM events WHERE payload ? 'errorCode';

-- JSONB operators
SELECT payload->'user'->>'name' AS user_name FROM events;
SELECT payload #>> '{address,city}' AS city FROM events;

-- GIN index for JSONB
CREATE INDEX idx_events_payload ON events USING GIN(payload);
CREATE INDEX idx_events_payload_path ON events USING GIN(payload jsonb_path_ops);

-- JSONB aggregation
SELECT user_id, jsonb_agg(jsonb_build_object('id', id, 'type', type)) AS events
FROM events GROUP BY user_id;
```

## Upsert (INSERT ON CONFLICT)

```sql
INSERT INTO products (sku, name, price, stock)
VALUES ('WDG-001', 'Widget', 9.99, 100)
ON CONFLICT (sku) DO UPDATE SET
  price = EXCLUDED.price,
  stock = products.stock + EXCLUDED.stock,
  updated_at = now();
```

## References

- [Query Patterns](references/query-patterns.md) — Pivot tables, gap analysis, running totals, de-duplication, hierarchical queries.
- [PostgreSQL Features](references/postgresql-features.md) — Array types, custom types, extensions, RLS, partitioning.
