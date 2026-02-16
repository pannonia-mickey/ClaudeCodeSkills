---
name: Database Security
description: This skill should be used when the user asks about "SQL injection", "database access control", "database encryption", "parameterized queries", "database audit logging", "GRANT REVOKE", "database secrets rotation", or "database security". It covers SQL injection prevention, access control, encryption, and audit logging.
---

# Database Security

## SQL Injection Prevention

```sql
-- ALWAYS use parameterized queries
-- BAD (SQL injection vulnerable)
query = f"SELECT * FROM users WHERE email = '{email}'"

-- GOOD — Parameterized (every language/ORM supports this)
-- PostgreSQL: $1, $2
SELECT * FROM users WHERE email = $1;
-- MySQL: ?
SELECT * FROM users WHERE email = ?;
```

## Access Control (Principle of Least Privilege)

```sql
-- Create application role with minimal permissions
CREATE ROLE app_user LOGIN PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Read-only role for reporting
CREATE ROLE report_reader LOGIN PASSWORD 'report_password';
GRANT CONNECT ON DATABASE mydb TO report_reader;
GRANT USAGE ON SCHEMA public TO report_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO report_reader;

-- NEVER use superuser for application connections
-- REVOKE dangerous permissions
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;
```

## Encryption

```sql
-- Encryption at rest: Enable in database configuration
-- PostgreSQL: Use OS-level disk encryption or TDE

-- Encryption in transit: Require SSL
-- postgresql.conf: ssl = on
-- pg_hba.conf: hostssl all all 0.0.0.0/0 scram-sha-256

-- Column-level encryption with pgcrypto
CREATE EXTENSION pgcrypto;

-- Encrypt sensitive data
INSERT INTO users (email, ssn_encrypted)
VALUES ('alice@test.com', pgp_sym_encrypt('123-45-6789', 'encryption_key'));

-- Decrypt
SELECT email, pgp_sym_decrypt(ssn_encrypted, 'encryption_key') AS ssn
FROM users WHERE id = $1;
```

## Password Hashing

```sql
-- Use pgcrypto for password hashing
CREATE EXTENSION pgcrypto;

-- Hash password (bcrypt)
INSERT INTO users (email, password_hash)
VALUES ($1, crypt($2, gen_salt('bf', 12)));

-- Verify password
SELECT id FROM users
WHERE email = $1 AND password_hash = crypt($2, password_hash);
```

## Audit Logging

```sql
-- See database-mastery/references/schema-patterns.md for full audit trigger

-- Query audit log
SELECT * FROM audit_log
WHERE table_name = 'users'
  AND action = 'UPDATE'
  AND changed_at > now() - INTERVAL '24 hours'
ORDER BY changed_at DESC;
```

## References

- [Access Control Patterns](references/access-control-patterns.md) — RLS, role hierarchy, connection security, pg_hba.conf.
- [Compliance Patterns](references/compliance-patterns.md) — PII handling, data retention, GDPR right to erasure, encryption strategies.
