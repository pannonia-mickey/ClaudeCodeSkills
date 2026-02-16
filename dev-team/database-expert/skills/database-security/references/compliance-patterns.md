# Database Compliance Patterns

## PII Handling

```sql
-- Identify PII columns
COMMENT ON COLUMN users.email IS 'PII: email address';
COMMENT ON COLUMN users.phone IS 'PII: phone number';
COMMENT ON COLUMN users.ssn_encrypted IS 'PII: SSN (encrypted)';

-- Mask PII in non-production environments
CREATE VIEW users_masked AS
SELECT
  id,
  regexp_replace(email, '(.).*@', '\1***@') AS email,
  regexp_replace(phone, '\d(?=\d{4})', '*') AS phone,
  name,
  created_at
FROM users;

-- Column-level encryption for sensitive data
-- See database-security/SKILL.md for pgcrypto examples
```

## Data Retention

```sql
-- Automated data retention with partitioning
CREATE TABLE audit_log (
  id BIGSERIAL,
  action VARCHAR(50),
  data JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create partitions by month
-- Drop partitions older than retention period
DROP TABLE IF EXISTS audit_log_2024_01; -- Data older than 1 year

-- Automated cleanup job
CREATE OR REPLACE FUNCTION cleanup_expired_data() RETURNS void AS $$
BEGIN
  -- Delete expired sessions
  DELETE FROM sessions WHERE expires_at < now();

  -- Delete soft-deleted records after 30 days
  DELETE FROM users WHERE deleted_at < now() - INTERVAL '30 days';

  -- Archive old orders
  INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < now() - INTERVAL '2 years';
  DELETE FROM orders WHERE created_at < now() - INTERVAL '2 years';
END;
$$ LANGUAGE plpgsql;

-- Schedule with pg_cron
SELECT cron.schedule('cleanup-expired', '0 3 * * *', 'SELECT cleanup_expired_data()');
```

## GDPR Right to Erasure

```sql
-- Anonymize user data instead of hard delete (preserves referential integrity)
CREATE OR REPLACE FUNCTION anonymize_user(target_user_id UUID) RETURNS void AS $$
BEGIN
  UPDATE users SET
    email = 'deleted_' || target_user_id || '@anonymized.local',
    name = 'Deleted User',
    phone = NULL,
    address = NULL,
    deleted_at = now(),
    anonymized_at = now()
  WHERE id = target_user_id;

  -- Remove from search indexes
  UPDATE users SET search_vector = NULL WHERE id = target_user_id;

  -- Log the erasure request
  INSERT INTO data_erasure_log (user_id, requested_at, completed_at)
  VALUES (target_user_id, now(), now());
END;
$$ LANGUAGE plpgsql;
```

## Encryption at Rest

```sql
-- Option 1: Filesystem encryption (recommended)
-- Use LUKS/dm-crypt on Linux, BitLocker on Windows, EBS encryption on AWS

-- Option 2: Transparent Data Encryption (TDE)
-- Available in PostgreSQL via extensions or managed services (RDS, Azure)

-- Option 3: Column-level encryption (for specific sensitive fields)
-- Use pgcrypto for encrypting individual columns
-- Key management: Store encryption keys in external vault (AWS KMS, HashiCorp Vault)
```

## Backup Security

```bash
# Encrypted backups
pg_dump mydb | gpg --encrypt --recipient backup@company.com > backup.sql.gpg

# Verify backup integrity
gpg --decrypt backup.sql.gpg | head -5

# Secure backup storage
# - Encrypt at rest
# - Restrict access (IAM roles, filesystem permissions)
# - Test restores regularly
# - Store in different region/account than production
```

## Compliance Checklist

- [ ] PII columns identified and documented
- [ ] Sensitive data encrypted at rest and in transit
- [ ] Access control follows least privilege principle
- [ ] Audit logging enabled for all data modifications
- [ ] Data retention policies implemented and automated
- [ ] GDPR erasure/anonymization procedure tested
- [ ] Backups encrypted and tested regularly
- [ ] Database connections use SSL/TLS
- [ ] No superuser credentials in application code
- [ ] Regular access reviews and credential rotation
