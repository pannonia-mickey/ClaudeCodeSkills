# Test Patterns

## Test Factories

```typescript
import { faker } from '@faker-js/faker';

function createUser(overrides: Partial<User> = {}): User {
  return {
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: 'user',
    createdAt: faker.date.recent(),
    ...overrides,
  };
}

function createOrder(overrides: Partial<Order> = {}): Order {
  return {
    id: faker.string.uuid(),
    userId: faker.string.uuid(),
    items: [{ productId: faker.string.uuid(), quantity: 1, price: faker.number.float({ min: 1, max: 100 }) }],
    status: 'pending',
    ...overrides,
  };
}

// Builder for complex objects
class OrderBuilder {
  private data: Partial<Order> = {};
  withItems(count: number) { this.data.items = Array.from({ length: count }, () => createOrderItem()); return this; }
  withStatus(status: string) { this.data.status = status; return this; }
  withUser(userId: string) { this.data.userId = userId; return this; }
  build() { return createOrder(this.data); }
}
```

## Custom Matchers

```typescript
// Vitest custom matcher
expect.extend({
  toBeValidUUID(received: string) {
    const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    return { pass: uuidRegex.test(received), message: () => `expected ${received} to be a valid UUID` };
  },
  toBeWithinMs(received: number, expected: number, tolerance: number) {
    const pass = Math.abs(received - expected) <= tolerance;
    return { pass, message: () => `expected ${received} to be within ${tolerance}ms of ${expected}` };
  },
});

// Usage
expect(user.id).toBeValidUUID();
expect(responseTime).toBeWithinMs(200, 50);
```

## Parameterized Tests

```typescript
describe.each([
  { input: '', expected: false, desc: 'empty string' },
  { input: 'a@b.c', expected: true, desc: 'simple email' },
  { input: 'no-at-sign', expected: false, desc: 'missing @' },
  { input: 'user@.com', expected: false, desc: 'missing domain' },
  { input: 'user+tag@domain.com', expected: true, desc: 'with plus tag' },
])('isValidEmail("$input") - $desc', ({ input, expected }) => {
  it(`should return ${expected}`, () => {
    expect(isValidEmail(input)).toBe(expected);
  });
});
```

## Test Data Management

```typescript
// Seed function for integration tests
async function seedTestData(db: Database) {
  const admin = await db.users.create(createUser({ role: 'admin' }));
  const user = await db.users.create(createUser());
  const products = await Promise.all(
    Array.from({ length: 5 }, () => db.products.create(createProduct()))
  );
  const order = await db.orders.create(createOrder({
    userId: user.id,
    items: [{ productId: products[0].id, quantity: 2, price: products[0].price }],
  }));
  return { admin, user, products, order };
}

// Cleanup between tests
async function cleanupTestData(db: Database) {
  await db.execute('TRUNCATE users, products, orders CASCADE');
}
```

## Snapshot Testing

```typescript
// API response snapshot
it('should return correct error format', () => {
  const error = formatApiError(new NotFoundError('User'));
  expect(error).toMatchInlineSnapshot(`
    {
      "code": "NOT_FOUND",
      "message": "User not found",
      "status": 404,
    }
  `);
});

// Ignore dynamic fields
expect(response).toMatchSnapshot({
  id: expect.any(String),
  createdAt: expect.any(String),
});
```
