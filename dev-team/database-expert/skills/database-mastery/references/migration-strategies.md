# Database Migration Strategies

## Zero-Downtime Migration Pattern

### Adding a Column

```sql
-- Safe: ADD COLUMN with default (PG 11+ is instant)
ALTER TABLE users ADD COLUMN role VARCHAR(20) NOT NULL DEFAULT 'user';

-- Unsafe: Adding NOT NULL without default on PG < 11 (rewrites table)
-- Instead: Add nullable → backfill → add constraint
ALTER TABLE users ADD COLUMN role VARCHAR(20);
UPDATE users SET role = 'user' WHERE role IS NULL;
ALTER TABLE users ALTER COLUMN role SET NOT NULL;
ALTER TABLE users ALTER COLUMN role SET DEFAULT 'user';
```

### Renaming a Column

```sql
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);

-- Step 2: Backfill (in batches for large tables)
UPDATE users SET full_name = name WHERE full_name IS NULL AND id > $last_id LIMIT 10000;

-- Step 3: Update application to write both columns
-- Step 4: Update application to read from new column
-- Step 5: Drop old column
ALTER TABLE users DROP COLUMN name;
```

### Removing a Column

```sql
-- Step 1: Stop writing to column in application
-- Step 2: Stop reading from column in application
-- Step 3: Drop column
ALTER TABLE users DROP COLUMN legacy_field;
```

### Adding an Index

```sql
-- Use CONCURRENTLY to avoid locking
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
-- Note: Cannot run inside a transaction
```

## Batch Backfill Pattern

```sql
-- Backfill in batches to avoid long-running transactions
DO $$
DECLARE
  batch_size INT := 10000;
  affected INT := 1;
BEGIN
  WHILE affected > 0 LOOP
    UPDATE users
    SET normalized_email = lower(trim(email))
    WHERE normalized_email IS NULL
    AND id IN (
      SELECT id FROM users WHERE normalized_email IS NULL LIMIT batch_size
    );
    GET DIAGNOSTICS affected = ROW_COUNT;
    RAISE NOTICE 'Updated % rows', affected;
    COMMIT;
    PERFORM pg_sleep(0.1); -- Small pause to reduce load
  END LOOP;
END $$;
```

## Schema Versioning

```
migrations/
  001_create_users.sql
  002_create_orders.sql
  003_add_user_role.sql
  004_add_orders_index.sql
```

```sql
-- Migration tracking table
CREATE TABLE schema_migrations (
  version VARCHAR(255) PRIMARY KEY,
  applied_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Rollback Patterns

```sql
-- Reversible migration pair
-- UP
ALTER TABLE products ADD COLUMN category_id UUID REFERENCES categories(id);
CREATE INDEX idx_products_category ON products(category_id);

-- DOWN
DROP INDEX IF EXISTS idx_products_category;
ALTER TABLE products DROP COLUMN IF EXISTS category_id;
```

## Large Table Migrations

```sql
-- For tables with millions of rows, use pg_repack or create-swap pattern

-- Create new table with desired schema
CREATE TABLE users_new (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(254) NOT NULL,
  name VARCHAR(100) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'user',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Copy data in batches
INSERT INTO users_new (id, email, name, created_at)
SELECT id, email, name, created_at FROM users
WHERE id > $cursor ORDER BY id LIMIT 50000;

-- Swap (brief lock)
BEGIN;
  ALTER TABLE users RENAME TO users_old;
  ALTER TABLE users_new RENAME TO users;
COMMIT;

-- Drop old table after verification
DROP TABLE users_old;
```
