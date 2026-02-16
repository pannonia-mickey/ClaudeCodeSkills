# Node.js Integration Testing

## Testcontainers

```typescript
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { RedisContainer, StartedRedisContainer } from '@testcontainers/redis';
import { PrismaClient } from '@prisma/client';
import { execSync } from 'node:child_process';

let pgContainer: StartedPostgreSqlContainer;
let redisContainer: StartedRedisContainer;
let prisma: PrismaClient;

beforeAll(async () => {
  // Start containers
  pgContainer = await new PostgreSqlContainer('postgres:16').start();
  redisContainer = await new RedisContainer().start();

  // Set env for Prisma
  process.env.DATABASE_URL = pgContainer.getConnectionUri();
  process.env.REDIS_URL = redisContainer.getConnectionUrl();

  // Run migrations
  execSync('npx prisma migrate deploy', { env: process.env });

  prisma = new PrismaClient();
}, 60_000); // 60s timeout for container startup

afterAll(async () => {
  await prisma.$disconnect();
  await pgContainer.stop();
  await redisContainer.stop();
});

afterEach(async () => {
  // Clean all tables between tests
  const tables = await prisma.$queryRaw<{ tablename: string }[]>`
    SELECT tablename FROM pg_tables WHERE schemaname = 'public'
  `;
  for (const { tablename } of tables) {
    if (tablename !== '_prisma_migrations') {
      await prisma.$executeRawUnsafe(`TRUNCATE TABLE "${tablename}" CASCADE`);
    }
  }
});
```

## Database Fixtures

```typescript
// fixtures/users.ts
import { faker } from '@faker-js/faker';

export function createUserFixture(overrides: Partial<User> = {}): Prisma.UserCreateInput {
  return {
    email: faker.internet.email(),
    name: faker.person.fullName(),
    passwordHash: '$2b$10$fixedHashForTesting',
    role: 'user',
    ...overrides,
  };
}

// Seed helper
export async function seedUsers(prisma: PrismaClient, count = 5) {
  return Promise.all(
    Array.from({ length: count }, () =>
      prisma.user.create({ data: createUserFixture() })
    )
  );
}

// Usage in tests
describe('GET /api/users', () => {
  it('should return paginated users', async () => {
    await seedUsers(prisma, 25);

    const response = await request(app)
      .get('/api/users?page=1&pageSize=10')
      .expect(200);

    expect(response.body.data).toHaveLength(10);
    expect(response.body.meta.total).toBe(25);
  });
});
```

## Full Integration Test Suite

```typescript
describe('Order Flow (Integration)', () => {
  let authToken: string;
  let userId: string;

  beforeAll(async () => {
    // Create test user and authenticate
    const user = await prisma.user.create({ data: createUserFixture() });
    userId = user.id;
    authToken = generateToken({ sub: user.id, role: 'user' });
  });

  it('should create an order', async () => {
    // Seed product
    const product = await prisma.product.create({
      data: { name: 'Widget', price: 9.99, stock: 10 },
    });

    // Place order
    const response = await request(app)
      .post('/api/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ items: [{ productId: product.id, quantity: 2 }] })
      .expect(201);

    expect(response.body.data.total).toBe(19.98);

    // Verify stock updated
    const updated = await prisma.product.findUnique({ where: { id: product.id } });
    expect(updated!.stock).toBe(8);
  });

  it('should reject order for insufficient stock', async () => {
    const product = await prisma.product.create({
      data: { name: 'Rare Item', price: 99.99, stock: 1 },
    });

    await request(app)
      .post('/api/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ items: [{ productId: product.id, quantity: 5 }] })
      .expect(400);
  });
});
```

## Contract Testing with Pact

```typescript
import { PactV4, MatchersV3 } from '@pact-foundation/pact';

const provider = new PactV4({
  consumer: 'frontend',
  provider: 'users-api',
});

describe('Users API Contract', () => {
  it('should return user by id', async () => {
    await provider
      .addInteraction()
      .given('user 1 exists')
      .uponReceiving('a request for user 1')
      .withRequest('GET', '/api/users/1')
      .willRespondWith(200, (builder) => {
        builder.jsonBody({
          data: {
            id: MatchersV3.string('1'),
            name: MatchersV3.string('Alice'),
            email: MatchersV3.email(),
          },
        });
      })
      .executeTest(async (mockServer) => {
        const client = new UsersClient(mockServer.url);
        const user = await client.getUser('1');
        expect(user.name).toBe('Alice');
      });
  });
});
```

## CI Configuration

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm test -- --coverage

  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: test, POSTGRES_PASSWORD: test }
        ports: ['5432:5432']
        options: --health-cmd pg_isready
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx prisma migrate deploy
        env: { DATABASE_URL: 'postgresql://postgres:test@localhost:5432/test' }
      - run: npm run test:integration
        env:
          DATABASE_URL: 'postgresql://postgres:test@localhost:5432/test'
          REDIS_URL: 'redis://localhost:6379'
```
