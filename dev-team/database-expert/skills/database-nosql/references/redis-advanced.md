# Redis Advanced Patterns

## Pub/Sub

```typescript
// Publisher
await redis.publish('notifications', JSON.stringify({ userId: '123', message: 'New order' }));

// Subscriber
const sub = redis.duplicate();
await sub.subscribe('notifications');
sub.on('message', (channel, message) => {
  const data = JSON.parse(message);
  console.log(`[${channel}]`, data);
});

// Pattern subscription
await sub.psubscribe('events:*');
sub.on('pmessage', (pattern, channel, message) => {
  console.log(`[${channel}]`, message);
});
```

## Redis Streams

```typescript
// Producer
await redis.xadd('orders-stream', '*', { orderId: '123', action: 'created', total: '29.97' });

// Consumer group
await redis.xgroup('CREATE', 'orders-stream', 'order-processors', '0', 'MKSTREAM');

// Consumer
const messages = await redis.xreadgroup(
  'GROUP', 'order-processors', 'worker-1',
  'COUNT', 10,
  'BLOCK', 5000,
  'STREAMS', 'orders-stream', '>'
);

// Acknowledge processed message
await redis.xack('orders-stream', 'order-processors', messageId);
```

## Distributed Locking (Redlock)

```typescript
import Redlock from 'redlock';

const redlock = new Redlock([redis], {
  retryCount: 3,
  retryDelay: 200,
  retryJitter: 200,
});

// Acquire lock
const lock = await redlock.acquire(['lock:order:123'], 5000); // 5s TTL
try {
  // Critical section — only one process can execute this
  await processOrder('123');
} finally {
  await lock.release();
}
```

## Rate Limiting with Redis

```typescript
// Sliding window rate limiter
async function isRateLimited(userId: string, limit: number, windowMs: number): Promise<boolean> {
  const key = `rate:${userId}`;
  const now = Date.now();
  const windowStart = now - windowMs;

  const pipe = redis.pipeline();
  pipe.zremrangebyscore(key, 0, windowStart); // Remove old entries
  pipe.zadd(key, now, `${now}:${Math.random()}`); // Add current request
  pipe.zcard(key); // Count requests in window
  pipe.expire(key, Math.ceil(windowMs / 1000)); // Set TTL

  const results = await pipe.exec();
  const count = results![2][1] as number;
  return count > limit;
}
```

## Lua Scripting (Atomic Operations)

```typescript
// Atomic increment with limit
const script = `
  local current = tonumber(redis.call('GET', KEYS[1]) or '0')
  if current >= tonumber(ARGV[1]) then
    return -1
  end
  return redis.call('INCR', KEYS[1])
`;

const result = await redis.eval(script, 1, 'counter:api-calls', '1000');
if (result === -1) {
  throw new Error('Rate limit exceeded');
}
```

## Session Store

```typescript
import session from 'express-session';
import RedisStore from 'connect-redis';

app.use(session({
  store: new RedisStore({ client: redis, prefix: 'sess:' }),
  secret: config.sessionSecret,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,
    httpOnly: true,
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
  },
}));
```

## Cache Warming

```typescript
async function warmCache(): Promise<void> {
  // Warm popular products
  const popular = await db.products.findMany({
    orderBy: { viewCount: 'desc' },
    take: 100,
  });

  const pipeline = redis.pipeline();
  for (const product of popular) {
    pipeline.set(`product:${product.id}`, JSON.stringify(product), 'EX', 3600);
  }
  await pipeline.exec();
  console.log(`Warmed cache with ${popular.length} products`);
}
```
