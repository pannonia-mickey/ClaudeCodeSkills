# MongoDB Design Patterns

## Bucket Pattern (Time-Series)

```javascript
// Instead of one document per measurement:
// Group measurements into time-bucketed documents
{
  _id: ObjectId("..."),
  sensorId: "temp-001",
  bucket: ISODate("2025-01-15T10:00:00Z"), // Hour bucket
  measurements: [
    { ts: ISODate("2025-01-15T10:00:12Z"), value: 22.5 },
    { ts: ISODate("2025-01-15T10:01:03Z"), value: 22.6 },
    // ... up to ~200 entries per bucket
  ],
  count: 120,
  sum: 2700.5,
  min: 21.8,
  max: 23.2
}

// Benefits: fewer documents, pre-aggregated stats, efficient range queries
```

## Schema Versioning

```javascript
// Add version field to handle schema evolution
{
  _id: ObjectId("..."),
  schemaVersion: 2,
  name: "Alice Smith",  // v2: combined from firstName + lastName
  email: "alice@test.com"
}

// Migration function
function migrateUserToV2(doc) {
  if (doc.schemaVersion === 1) {
    return {
      ...doc,
      schemaVersion: 2,
      name: `${doc.firstName} ${doc.lastName}`,
    };
  }
  return doc;
}
```

## Computed Pattern

```javascript
// Pre-compute expensive calculations
{
  _id: ObjectId("..."),
  productId: "prod-123",
  reviews: [
    { userId: "user1", rating: 5, comment: "Great!" },
    { userId: "user2", rating: 4, comment: "Good" }
  ],
  // Computed fields — updated on every review add/update
  computed: {
    avgRating: 4.5,
    reviewCount: 2,
    ratingDistribution: { 1: 0, 2: 0, 3: 0, 4: 1, 5: 1 }
  }
}
```

## Subset Pattern

```javascript
// Store frequently accessed subset in main document
// Full data in separate collection

// products (hot data)
{
  _id: "prod-123",
  name: "Widget",
  price: 9.99,
  topReviews: [  // Only top 3 reviews embedded
    { rating: 5, comment: "Amazing!", userId: "user1" }
  ],
  reviewSummary: { avg: 4.5, count: 150 }
}

// reviews collection (cold data — full reviews)
{
  productId: "prod-123",
  userId: "user42",
  rating: 4,
  comment: "Detailed review text...",
  createdAt: ISODate("2025-01-15")
}
```

## MongoDB Indexes

```javascript
// Compound index (order matters for queries)
db.orders.createIndex({ userId: 1, createdAt: -1 });

// Text index for search
db.articles.createIndex({ title: "text", body: "text" });
db.articles.find({ $text: { $search: "database optimization" } });

// Partial index (index only matching documents)
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { active: true } }
);

// TTL index (auto-delete expired documents)
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });

// Wildcard index (JSONB-like flexibility)
db.events.createIndex({ "payload.$**": 1 });
```

## Transactions

```javascript
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    await orders.insertOne({ userId, items, total }, { session });
    await inventory.updateMany(
      { _id: { $in: items.map(i => i.productId) } },
      { $inc: { stock: -1 } },
      { session }
    );
    await users.updateOne(
      { _id: userId },
      { $inc: { orderCount: 1 } },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```
