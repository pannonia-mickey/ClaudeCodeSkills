---
name: Test Strategy & Planning
description: This skill should be used when the user asks about "test strategy", "coverage strategy", "risk-based testing", "contract testing", "Pact testing", "test data management", "test environment", "test pyramid", "test planning", or "test coverage targets". It covers test planning, coverage strategies, contract testing with Pact, and test data/environment management.
---

# Test Strategy & Planning

## Coverage Strategy

```typescript
// vitest.config.ts — coverage configuration
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      include: ['src/**/*.ts'],
      exclude: [
        'src/**/*.d.ts',
        'src/**/*.test.ts',
        'src/**/*.spec.ts',
        'src/**/index.ts', // barrel files
        'src/types/**',
      ],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      },
    },
  },
});
```

## Risk-Based Testing

```typescript
// Prioritize testing by risk level
// High risk: Payment, auth, data mutations → 90%+ coverage, integration + E2E
// Medium risk: Business logic, validations → 80%+ coverage, unit + integration
// Low risk: Display components, utilities → 60%+ coverage, unit tests

// High-risk: comprehensive test suite
describe('PaymentService', () => {
  // Unit tests for all paths
  it('should process valid payment', async () => { /* ... */ });
  it('should reject expired card', async () => { /* ... */ });
  it('should handle gateway timeout', async () => { /* ... */ });
  it('should handle idempotent retries', async () => { /* ... */ });

  // Integration tests with real dependencies
  it('should persist payment record', async () => { /* ... */ });
  it('should emit payment.completed event', async () => { /* ... */ });

  // Edge cases
  it('should handle concurrent duplicate payments', async () => { /* ... */ });
  it('should handle partial refunds correctly', async () => { /* ... */ });
});
```

## Contract Testing with Pact

```typescript
// Consumer-side contract test
import { PactV3, MatchersV3 } from '@pact-foundation/pact';

const provider = new PactV3({
  consumer: 'OrderService',
  provider: 'UserService',
});

describe('UserService contract', () => {
  it('should return user by ID', async () => {
    await provider
      .given('user with ID 123 exists')
      .uponReceiving('a request for user 123')
      .withRequest({
        method: 'GET',
        path: '/api/users/123',
        headers: { Authorization: MatchersV3.string('Bearer token') },
      })
      .willRespondWith({
        status: 200,
        body: MatchersV3.like({
          id: '123',
          name: MatchersV3.string('Alice'),
          email: MatchersV3.email(),
        }),
      })
      .executeTest(async (mockServer) => {
        const client = new UserClient(mockServer.url);
        const user = await client.getUser('123');
        expect(user.id).toBe('123');
        expect(user.name).toBeDefined();
      });
  });
});
```

## Test Data Management

```typescript
// Factory pattern with relationships
import { faker } from '@faker-js/faker';

class TestDataBuilder {
  private db: Database;

  async createUserWithOrders(orderCount = 3) {
    const user = await this.db.users.create({
      name: faker.person.fullName(),
      email: faker.internet.email(),
    });

    const orders = await Promise.all(
      Array.from({ length: orderCount }, () =>
        this.db.orders.create({
          userId: user.id,
          total: faker.number.float({ min: 10, max: 500 }),
          status: faker.helpers.arrayElement(['pending', 'shipped', 'delivered']),
        })
      )
    );

    return { user, orders };
  }

  async cleanup() {
    await this.db.execute('TRUNCATE users, orders CASCADE');
  }
}
```

## References

- [Coverage Strategies](references/coverage-strategies.md) — Risk-based coverage, mutation testing, coverage metrics, incremental coverage enforcement.
- [Contract Testing](references/contract-testing.md) — Pact provider verification, broker setup, CI integration, event-driven contracts.
