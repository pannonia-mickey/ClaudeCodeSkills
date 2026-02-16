# Database Schema Patterns

## Multi-Tenant Design

### Row-Level Security (PostgreSQL)

```sql
-- Shared tables with tenant column
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON projects
  USING (workspace_id = current_setting('app.workspace_id')::uuid);

-- Set context per request
SET app.workspace_id = '550e8400-e29b-41d4-a716-446655440000';
SELECT * FROM projects; -- Only returns projects for that workspace
```

### Schema-Per-Tenant

```sql
CREATE SCHEMA tenant_abc;
CREATE TABLE tenant_abc.projects (...);
-- Each tenant has isolated schema; more overhead, stronger isolation
```

## Audit Trail

```sql
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  table_name VARCHAR(100) NOT NULL,
  record_id UUID NOT NULL,
  action VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
  old_data JSONB,
  new_data JSONB,
  changed_by UUID REFERENCES users(id),
  changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Generic audit trigger
CREATE OR REPLACE FUNCTION audit_trigger_func() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_by)
  VALUES (
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    TG_OP,
    CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) END,
    CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) END,
    current_setting('app.user_id', true)::uuid
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_orders
  AFTER INSERT OR UPDATE OR DELETE ON orders
  FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

## Soft Deletes

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Partial index for active records
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;

-- Global query filter (application or view)
CREATE VIEW active_users AS
  SELECT * FROM users WHERE deleted_at IS NULL;
```

## Temporal Tables (System-Versioned)

```sql
-- PostgreSQL with temporal_tables extension or manual implementation
CREATE TABLE products (
  id UUID PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
  valid_to TIMESTAMPTZ NOT NULL DEFAULT 'infinity'
);

-- Get product price at a specific point in time
SELECT * FROM products
WHERE id = $1 AND valid_from <= $2 AND valid_to > $2;
```

## Polymorphic Associations

```sql
-- Option 1: Separate FK columns (type-safe)
CREATE TABLE comments (
  id UUID PRIMARY KEY,
  body TEXT NOT NULL,
  post_id UUID REFERENCES posts(id),
  product_id UUID REFERENCES products(id),
  CHECK (
    (post_id IS NOT NULL AND product_id IS NULL) OR
    (post_id IS NULL AND product_id IS NOT NULL)
  )
);

-- Option 2: Generic relation (flexible but no FK)
CREATE TABLE comments (
  id UUID PRIMARY KEY,
  body TEXT NOT NULL,
  commentable_type VARCHAR(50) NOT NULL,
  commentable_id UUID NOT NULL
);
CREATE INDEX idx_comments_target ON comments(commentable_type, commentable_id);
```

## Auto-Updated Timestamps

```sql
CREATE OR REPLACE FUNCTION update_updated_at() RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```
