---
name: NoSQL Databases
description: This skill should be used when the user asks about "MongoDB schema", "Redis caching", "DynamoDB design", "document database", "key-value store", "MongoDB aggregation", "Redis data structures", "single-table design", or "when to use NoSQL". It covers MongoDB document design, Redis patterns, and DynamoDB modeling.
---

# NoSQL Databases

## MongoDB Document Design

### Embedding vs Referencing

```javascript
// Embedding (denormalized) — when data is always accessed together
{
  _id: ObjectId("..."),
  name: "Alice",
  address: {             // Embedded — 1:1, always needed with user
    street: "123 Main St",
    city: "Springfield",
    state: "IL"
  },
  orders: [              // Embedded — 1:few, bounded growth
    { productId: "...", quantity: 2, price: 9.99 }
  ]
}

// Referencing (normalized) — when data is shared or grows unbounded
// users collection
{ _id: ObjectId("user1"), name: "Alice" }

// orders collection (references user)
{ _id: ObjectId("order1"), userId: ObjectId("user1"), items: [...], total: 29.97 }
```

### Aggregation Pipeline

```javascript
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2025-01-01") } } },
  { $unwind: "$items" },
  { $group: {
    _id: "$items.productId",
    totalRevenue: { $sum: { $multiply: ["$items.price", "$items.quantity"] } },
    totalSold: { $sum: "$items.quantity" },
    orderCount: { $sum: 1 }
  }},
  { $sort: { totalRevenue: -1 } },
  { $limit: 10 },
  { $lookup: {
    from: "products",
    localField: "_id",
    foreignField: "_id",
    as: "product"
  }},
  { $project: {
    productName: { $arrayElemAt: ["$product.name", 0] },
    totalRevenue: 1,
    totalSold: 1
  }}
]);
```

## Redis Data Structures

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

// String — simple cache
await redis.set('user:123', JSON.stringify(user), 'EX', 3600);
const cached = JSON.parse(await redis.get('user:123') ?? 'null');

// Hash — structured data
await redis.hset('user:123', { name: 'Alice', email: 'alice@test.com', role: 'admin' });
const name = await redis.hget('user:123', 'name');
const all = await redis.hgetall('user:123');

// Sorted Set — leaderboard, ranking
await redis.zadd('leaderboard', 1500, 'player1', 2000, 'player2', 1800, 'player3');
const top10 = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');

// List — queue, recent items
await redis.lpush('notifications:user1', JSON.stringify(notification));
const recent = await redis.lrange('notifications:user1', 0, 9);

// Set — unique items, tags
await redis.sadd('product:123:tags', 'electronics', 'sale', 'featured');
const tags = await redis.smembers('product:123:tags');
const inBoth = await redis.sinter('product:123:tags', 'product:456:tags');
```

## Redis Caching Patterns

```typescript
// Cache-aside
async function getUser(id: string): Promise<User> {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.users.findById(id);
  if (user) await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 3600);
  return user;
}

// Write-through
async function updateUser(id: string, data: Partial<User>): Promise<User> {
  const user = await db.users.update(id, data);
  await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 3600);
  return user;
}

// Cache invalidation
async function deleteUser(id: string): Promise<void> {
  await db.users.delete(id);
  await redis.del(`user:${id}`);
}
```

## DynamoDB Single-Table Design

```typescript
// All entities in one table, differentiated by PK/SK pattern
// PK: partition key, SK: sort key
{
  PK: "USER#123",  SK: "PROFILE",     name: "Alice", email: "alice@test.com"
  PK: "USER#123",  SK: "ORDER#001",   total: 29.97, status: "completed"
  PK: "USER#123",  SK: "ORDER#002",   total: 15.50, status: "pending"
  PK: "PRODUCT#A", SK: "DETAILS",     name: "Widget", price: 9.99
  PK: "PRODUCT#A", SK: "REVIEW#001",  rating: 5, comment: "Great!"
}

// Query patterns:
// Get user profile: PK = "USER#123", SK = "PROFILE"
// Get user's orders: PK = "USER#123", SK begins_with "ORDER#"
// Get product + reviews: PK = "PRODUCT#A"
```

## References

- [MongoDB Patterns](references/mongodb-patterns.md) — Schema versioning, bucket pattern, computed pattern, subset pattern.
- [Redis Advanced](references/redis-advanced.md) — Pub/Sub, Streams, Lua scripting, distributed locks, rate limiting.
