# Database Scaling Patterns

## Vertical Scaling

- Increase CPU, RAM, storage on existing server
- Simplest approach, limited ceiling
- PostgreSQL benefits: more shared_buffers, work_mem, effective_cache_size

## Read Replicas

```
┌──────────┐     Writes      ┌──────────┐
│  App     │ ───────────────> │ Primary  │
│  Server  │                  │   DB     │
│          │     Reads        └────┬─────┘
│          │ <─────────────────┐   │ Replication
│          │                   │   │
│          │     Reads        ┌┴───▼─────┐
│          │ <───────────────>│ Replica 1│
│          │                  └──────────┘
│          │     Reads        ┌──────────┐
│          │ <───────────────>│ Replica 2│
└──────────┘                  └──────────┘
```

```typescript
// Application-level routing
class DatabaseRouter {
  constructor(
    private primary: PrismaClient,
    private replicas: PrismaClient[],
  ) {}

  write() { return this.primary; }

  read() {
    // Round-robin across replicas
    const idx = Math.floor(Math.random() * this.replicas.length);
    return this.replicas[idx];
  }
}

// Usage
const users = await db.read().user.findMany();
const newUser = await db.write().user.create({ data: input });
```

## Sharding Strategies

### Range-Based

```
Shard 1: users A-M
Shard 2: users N-Z
```

### Hash-Based

```
Shard = hash(user_id) % num_shards
```

### Geography-Based

```
Shard US: US customers
Shard EU: EU customers
Shard APAC: APAC customers
```

## Connection Pooling Architecture

```
App Servers (100+ connections)
        │
        ▼
  PgBouncer (pool)
        │
        ▼
  PostgreSQL (20 connections)
```

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
```

## Caching Layer

```
Request → Check Cache → Hit? → Return cached
                    │
                    └─ Miss? → Query DB → Store in Cache → Return
```

```typescript
// Multi-level caching
class CacheService {
  constructor(
    private l1: Map<string, { data: unknown; exp: number }>, // In-memory
    private l2: Redis, // Redis
  ) {}

  async get<T>(key: string): Promise<T | null> {
    // L1 check (fastest)
    const l1 = this.l1.get(key);
    if (l1 && l1.exp > Date.now()) return l1.data as T;

    // L2 check (fast)
    const l2 = await this.l2.get(key);
    if (l2) {
      const data = JSON.parse(l2) as T;
      this.l1.set(key, { data, exp: Date.now() + 60_000 }); // L1: 1 min
      return data;
    }

    return null;
  }

  async set<T>(key: string, data: T, ttl: number): Promise<void> {
    this.l1.set(key, { data, exp: Date.now() + Math.min(ttl, 60_000) });
    await this.l2.set(key, JSON.stringify(data), 'PX', ttl);
  }
}
```

## Monitoring Queries

```sql
-- Active connections
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - pg_stat_activity.query_start > interval '5 seconds';

-- Table sizes
SELECT
  relname AS table_name,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
  pg_size_pretty(pg_relation_size(relid)) AS data_size,
  pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```
