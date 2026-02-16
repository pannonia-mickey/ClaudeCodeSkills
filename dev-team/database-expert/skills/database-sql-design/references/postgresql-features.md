# PostgreSQL Advanced Features

## Array Types

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  tags TEXT[] NOT NULL DEFAULT '{}',
  prices DECIMAL(10,2)[]
);

-- Query arrays
SELECT * FROM products WHERE 'electronics' = ANY(tags);
SELECT * FROM products WHERE tags @> ARRAY['electronics', 'sale'];
SELECT * FROM products WHERE tags && ARRAY['electronics', 'clothing']; -- overlap

-- GIN index for array containment
CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- Array aggregation
SELECT category, array_agg(DISTINCT tag ORDER BY tag) AS all_tags
FROM products, unnest(tags) AS tag
GROUP BY category;
```

## Custom Types

```sql
-- Enum types
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled');

-- Composite types
CREATE TYPE address AS (
  street VARCHAR(200),
  city VARCHAR(100),
  state VARCHAR(50),
  zip VARCHAR(20),
  country VARCHAR(2)
);

CREATE TABLE customers (
  id UUID PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  shipping_address address,
  billing_address address
);

SELECT (shipping_address).city FROM customers;
```

## Partitioning

```sql
-- Range partitioning (time-series data)
CREATE TABLE events (
  id UUID DEFAULT gen_random_uuid(),
  type VARCHAR(50) NOT NULL,
  payload JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_q1 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
CREATE TABLE events_2025_q2 PARTITION OF events
  FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');

-- Auto-create partitions with pg_partman
CREATE EXTENSION pg_partman;
SELECT partman.create_parent('public.events', 'created_at', 'native', 'monthly');

-- List partitioning
CREATE TABLE orders (...) PARTITION BY LIST (region);
CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('EU');

-- Hash partitioning (distribute evenly)
CREATE TABLE sessions (...) PARTITION BY HASH (user_id);
CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

## Full-Text Search

```sql
-- Add search vector column
ALTER TABLE articles ADD COLUMN search_vector tsvector;

-- Populate and keep updated
CREATE OR REPLACE FUNCTION update_search_vector() RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.body, '')), 'B');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER search_vector_update
  BEFORE INSERT OR UPDATE ON articles
  FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- GIN index
CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);

-- Search with ranking
SELECT id, title,
  ts_rank(search_vector, query) AS rank,
  ts_headline('english', body, query, 'MaxWords=50, MinWords=20') AS snippet
FROM articles, plainto_tsquery('english', 'database optimization') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

## Row-Level Security (RLS)

```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Users can only see their own documents
CREATE POLICY user_documents ON documents
  FOR ALL
  USING (owner_id = current_setting('app.user_id')::uuid);

-- Admins can see everything
CREATE POLICY admin_all ON documents
  FOR ALL
  USING (current_setting('app.user_role') = 'admin');

-- Set context in application
SET LOCAL app.user_id = '...';
SET LOCAL app.user_role = 'user';
```
