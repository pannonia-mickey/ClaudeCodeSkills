# Node.js API Best Practices

## Pagination

### Cursor-Based (Recommended for large datasets)

```typescript
// Request: GET /api/posts?cursor=abc123&limit=20
// Response:
{
  "data": [...],
  "meta": {
    "nextCursor": "xyz789",
    "hasMore": true
  }
}

// Implementation with Prisma
async function getPosts(cursor?: string, limit = 20) {
  const posts = await prisma.post.findMany({
    take: limit + 1, // Fetch one extra to check hasMore
    ...(cursor && { cursor: { id: cursor }, skip: 1 }),
    orderBy: { createdAt: 'desc' },
  });

  const hasMore = posts.length > limit;
  const data = hasMore ? posts.slice(0, -1) : posts;

  return {
    data,
    meta: { nextCursor: data.at(-1)?.id, hasMore },
  };
}
```

### Offset-Based (Simple, good for small datasets)

```typescript
// Request: GET /api/posts?page=2&pageSize=20
async function getPosts(page = 1, pageSize = 20) {
  const [data, total] = await Promise.all([
    prisma.post.findMany({ skip: (page - 1) * pageSize, take: pageSize }),
    prisma.post.count(),
  ]);
  return { data, meta: { page, pageSize, total, totalPages: Math.ceil(total / pageSize) } };
}
```

## Filtering and Sorting

```typescript
// GET /api/products?category=electronics&minPrice=10&maxPrice=100&sort=-price,name
const FilterSchema = z.object({
  category: z.string().optional(),
  minPrice: z.coerce.number().optional(),
  maxPrice: z.coerce.number().optional(),
  sort: z.string().optional(), // "-price,name" → price DESC, name ASC
});

function buildWhere(filters: z.infer<typeof FilterSchema>) {
  return {
    ...(filters.category && { category: filters.category }),
    ...(filters.minPrice && { price: { gte: filters.minPrice } }),
    ...(filters.maxPrice && { price: { lte: filters.maxPrice } }),
  };
}

function buildOrderBy(sort?: string) {
  if (!sort) return { createdAt: 'desc' as const };
  return sort.split(',').map(field => {
    const desc = field.startsWith('-');
    return { [field.replace('-', '')]: desc ? 'desc' as const : 'asc' as const };
  });
}
```

## API Versioning

```typescript
// URL versioning (most common)
app.use('/api/v1/users', v1UsersRouter);
app.use('/api/v2/users', v2UsersRouter);

// Header versioning
app.use('/api/users', (req, res, next) => {
  const version = req.headers['api-version'] || '1';
  if (version === '2') return v2UsersRouter(req, res, next);
  return v1UsersRouter(req, res, next);
});
```

## Response Envelope

```typescript
// Consistent response format
interface ApiResponse<T> {
  data: T;
  meta?: Record<string, unknown>;
}

interface ApiError {
  error: {
    code: string;
    message: string;
    details?: unknown;
  };
}

// Success
res.json({ data: user });
res.json({ data: users, meta: { page: 1, total: 100 } });

// Error
res.status(404).json({ error: { code: 'NOT_FOUND', message: 'User not found' } });
res.status(400).json({ error: { code: 'VALIDATION', message: 'Invalid input', details: fieldErrors } });
```

## Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });

// Global rate limit
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redis.sendCommand(args) }),
}));

// Stricter limit for auth endpoints
app.use('/api/auth', rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }));
```

## OpenAPI Generation

```typescript
// With @nestjs/swagger
@ApiTags('users')
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'List all users' })
  @ApiResponse({ status: 200, type: [UserDto] })
  findAll() {}
}

// With Fastify (auto-generated from JSON Schema)
fastify.get('/users', {
  schema: {
    response: { 200: { type: 'array', items: { $ref: 'User' } } },
  },
}, handler);
await app.register(fastifySwagger, { openapi: { info: { title: 'My API', version: '1.0' } } });
```
