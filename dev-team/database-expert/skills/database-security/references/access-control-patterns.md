# Database Access Control Patterns

## Role Hierarchy

```sql
-- Base roles (no login)
CREATE ROLE readonly;
CREATE ROLE readwrite;
CREATE ROLE admin;

-- Grant permissions to base roles
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

GRANT readonly TO readwrite;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO readwrite;

GRANT readwrite TO admin;
GRANT CREATE ON SCHEMA public TO admin;
GRANT TRUNCATE ON ALL TABLES IN SCHEMA public TO admin;

-- Login roles inherit from base roles
CREATE ROLE app_service LOGIN PASSWORD '...' IN ROLE readwrite;
CREATE ROLE report_service LOGIN PASSWORD '...' IN ROLE readonly;
CREATE ROLE admin_user LOGIN PASSWORD '...' IN ROLE admin;

-- Auto-grant on new tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT INSERT, UPDATE, DELETE ON TABLES TO readwrite;
```

## pg_hba.conf (Connection Security)

```
# TYPE  DATABASE  USER       ADDRESS         METHOD
# Local connections
local   all       postgres                   peer
local   all       all                        scram-sha-256

# SSL connections only from app servers
hostssl mydb      app_service  10.0.1.0/24   scram-sha-256
hostssl mydb      report_svc   10.0.2.0/24   scram-sha-256

# Reject all others
host    all       all          0.0.0.0/0     reject
```

## Row-Level Security (RLS)

```sql
-- Multi-tenant isolation
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents FORCE ROW LEVEL SECURITY; -- Apply even to table owners

-- Policy: users see only their organization's documents
CREATE POLICY org_isolation ON documents
  FOR ALL
  USING (org_id = current_setting('app.org_id')::uuid)
  WITH CHECK (org_id = current_setting('app.org_id')::uuid);

-- Policy: admins bypass isolation
CREATE POLICY admin_access ON documents
  FOR ALL
  USING (current_setting('app.role') = 'admin');

-- Set context per request (in application code)
BEGIN;
  SELECT set_config('app.org_id', $1, true); -- true = local to transaction
  SELECT set_config('app.role', $2, true);
  -- All subsequent queries in this transaction are filtered by RLS
COMMIT;
```

## Connection Encryption

```sql
-- Verify SSL connection
SELECT ssl, version FROM pg_stat_ssl WHERE pid = pg_backend_pid();

-- Force SSL for all connections
-- postgresql.conf
-- ssl = on
-- ssl_cert_file = '/path/to/server.crt'
-- ssl_key_file = '/path/to/server.key'
-- ssl_ca_file = '/path/to/ca.crt'

-- Application connection string
-- postgresql://user:pass@host/db?sslmode=verify-full&sslrootcert=ca.crt
```

## Monitoring Access

```sql
-- Enable query logging
-- postgresql.conf: log_statement = 'mod' (logs INSERT/UPDATE/DELETE)
-- postgresql.conf: log_connections = on
-- postgresql.conf: log_disconnections = on

-- Check active connections
SELECT usename, client_addr, application_name, state, query
FROM pg_stat_activity
WHERE usename != 'postgres';

-- Check failed login attempts (from logs)
-- grep "FATAL.*password authentication failed" /var/log/postgresql/postgresql-*.log
```

## Database Firewall Rules

```sql
-- PostgreSQL extension: pgaudit for detailed audit logging
CREATE EXTENSION pgaudit;
-- postgresql.conf: pgaudit.log = 'write, ddl'

-- Log all DDL and write operations
ALTER SYSTEM SET pgaudit.log = 'write, ddl, role';
SELECT pg_reload_conf();
```
