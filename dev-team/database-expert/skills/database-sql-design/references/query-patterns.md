# SQL Query Patterns

## Pivot Table

```sql
SELECT
  product_id,
  SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN amount ELSE 0 END) AS jan,
  SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 2 THEN amount ELSE 0 END) AS feb,
  SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 3 THEN amount ELSE 0 END) AS mar
FROM order_items
WHERE EXTRACT(YEAR FROM order_date) = 2025
GROUP BY product_id;

-- PostgreSQL crosstab (tablefunc extension)
SELECT * FROM crosstab(
  'SELECT product_id, EXTRACT(MONTH FROM order_date)::int, SUM(amount)
   FROM order_items GROUP BY 1, 2 ORDER BY 1, 2'
) AS ct(product_id UUID, jan DECIMAL, feb DECIMAL, mar DECIMAL);
```

## Gap Analysis (Find Missing Sequences)

```sql
-- Find gaps in sequential IDs
SELECT prev_id + 1 AS gap_start, id - 1 AS gap_end
FROM (
  SELECT id, LAG(id) OVER (ORDER BY id) AS prev_id
  FROM invoices
) t
WHERE id - prev_id > 1;

-- Find date gaps
WITH date_series AS (
  SELECT generate_series(
    (SELECT MIN(date) FROM daily_metrics),
    (SELECT MAX(date) FROM daily_metrics),
    '1 day'::interval
  )::date AS date
)
SELECT ds.date AS missing_date
FROM date_series ds
LEFT JOIN daily_metrics dm ON dm.date = ds.date
WHERE dm.date IS NULL;
```

## De-Duplication

```sql
-- Keep latest, delete duplicates
DELETE FROM users
WHERE id NOT IN (
  SELECT DISTINCT ON (email) id
  FROM users
  ORDER BY email, created_at DESC
);

-- Using window function
WITH ranked AS (
  SELECT id, email,
    ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC) AS rn
  FROM users
)
DELETE FROM users WHERE id IN (SELECT id FROM ranked WHERE rn > 1);
```

## Pagination Patterns

```sql
-- Cursor-based (performant for large datasets)
SELECT id, name, created_at
FROM products
WHERE created_at < $cursor_timestamp
  OR (created_at = $cursor_timestamp AND id < $cursor_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Offset-based with total count
SELECT *, COUNT(*) OVER() AS total_count
FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;

-- Keyset pagination with LATERAL
SELECT p.* FROM (
  SELECT created_at, id FROM products
  WHERE (created_at, id) < ($last_created_at, $last_id)
  ORDER BY created_at DESC, id DESC
  LIMIT 20
) cursor
JOIN LATERAL (SELECT * FROM products WHERE id = cursor.id) p ON TRUE;
```

## Conditional Aggregation

```sql
SELECT
  COUNT(*) AS total_orders,
  COUNT(*) FILTER (WHERE status = 'completed') AS completed,
  COUNT(*) FILTER (WHERE status = 'pending') AS pending,
  SUM(amount) FILTER (WHERE status = 'completed') AS completed_revenue,
  AVG(amount) FILTER (WHERE created_at > now() - INTERVAL '30 days') AS avg_recent
FROM orders;
```

## Lateral Joins

```sql
-- Get top 3 recent orders per user
SELECT u.id, u.name, recent_orders.*
FROM users u
CROSS JOIN LATERAL (
  SELECT id AS order_id, total, created_at
  FROM orders
  WHERE user_id = u.id
  ORDER BY created_at DESC
  LIMIT 3
) recent_orders;
```

## Set Operations

```sql
-- Users who ordered but never reviewed
SELECT user_id FROM orders
EXCEPT
SELECT user_id FROM reviews;

-- Users who both ordered AND reviewed
SELECT user_id FROM orders
INTERSECT
SELECT user_id FROM reviews;
```
