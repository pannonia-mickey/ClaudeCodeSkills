# Database Optimization Techniques

## Query Rewriting

```sql
-- Rewrite correlated subquery as JOIN
-- SLOW
SELECT * FROM orders
WHERE total > (SELECT AVG(total) FROM orders WHERE user_id = orders.user_id);

-- FAST
SELECT o.*
FROM orders o
JOIN (SELECT user_id, AVG(total) AS avg_total FROM orders GROUP BY user_id) ua
  ON o.user_id = ua.user_id AND o.total > ua.avg_total;

-- Rewrite EXISTS vs IN
-- Use EXISTS for large outer, small inner
SELECT * FROM users u WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- Use IN for small outer, large inner (or let optimizer choose)
SELECT * FROM users WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- Avoid functions in WHERE (prevents index usage)
-- BAD
SELECT * FROM users WHERE YEAR(created_at) = 2025;
-- GOOD
SELECT * FROM users WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';
```

## Materialized Views

```sql
CREATE MATERIALIZED VIEW product_stats AS
SELECT
  p.id, p.name,
  COUNT(r.id) AS review_count,
  AVG(r.rating) AS avg_rating,
  SUM(oi.quantity) AS total_sold
FROM products p
LEFT JOIN reviews r ON r.product_id = p.id
LEFT JOIN order_items oi ON oi.product_id = p.id
GROUP BY p.id, p.name;

CREATE UNIQUE INDEX idx_product_stats_id ON product_stats(id);

-- Refresh (can be scheduled)
REFRESH MATERIALIZED VIEW CONCURRENTLY product_stats;

-- Query the view (instant, pre-computed)
SELECT * FROM product_stats WHERE avg_rating > 4.0 ORDER BY total_sold DESC;
```

## VACUUM and Statistics

```sql
-- Auto-vacuum settings (postgresql.conf)
-- autovacuum_vacuum_scale_factor = 0.1  (10% dead tuples triggers vacuum)
-- autovacuum_analyze_scale_factor = 0.05 (5% changes triggers analyze)

-- Manual vacuum for specific table
VACUUM ANALYZE orders;

-- Check table bloat
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Update statistics
ANALYZE users;

-- Check statistics
SELECT attname, n_distinct, most_common_vals
FROM pg_stats WHERE tablename = 'orders';
```

## Batch Processing

```sql
-- Process large datasets in batches to avoid lock contention
-- Delete old records in batches
DO $$
DECLARE
  deleted INT;
BEGIN
  LOOP
    DELETE FROM events
    WHERE id IN (
      SELECT id FROM events
      WHERE created_at < now() - INTERVAL '90 days'
      LIMIT 10000
    );
    GET DIAGNOSTICS deleted = ROW_COUNT;
    EXIT WHEN deleted = 0;
    PERFORM pg_sleep(0.5); -- Throttle
  END LOOP;
END $$;
```

## pg_stat_statements

```sql
-- Enable (postgresql.conf)
-- shared_preload_libraries = 'pg_stat_statements'

CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top slow queries
SELECT
  calls,
  ROUND(mean_exec_time::numeric, 2) AS avg_ms,
  ROUND(total_exec_time::numeric, 2) AS total_ms,
  rows,
  query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Reset stats
SELECT pg_stat_statements_reset();
```

## Index Maintenance

```sql
-- Find unused indexes
SELECT
  indexrelname AS index_name,
  relname AS table_name,
  idx_scan AS index_scans,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find missing indexes (tables with high sequential scans)
SELECT
  relname,
  seq_scan,
  idx_scan,
  ROUND(100.0 * seq_scan / NULLIF(seq_scan + idx_scan, 0), 1) AS seq_pct,
  n_live_tup
FROM pg_stat_user_tables
WHERE n_live_tup > 10000 AND seq_scan > idx_scan
ORDER BY seq_scan DESC;

-- Reindex (fix bloated indexes)
REINDEX INDEX CONCURRENTLY idx_orders_user_id;
```
