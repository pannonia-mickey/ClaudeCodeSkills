# Contract Testing

## Pact Provider Verification

```typescript
// Provider-side verification
import { Verifier } from '@pact-foundation/pact';

describe('UserService provider verification', () => {
  it('should validate consumer contracts', async () => {
    const verifier = new Verifier({
      providerBaseUrl: 'http://localhost:3001',
      provider: 'UserService',

      // Load contracts from broker
      pactBrokerUrl: 'https://pact-broker.internal',
      pactBrokerToken: process.env.PACT_BROKER_TOKEN,

      // Or from local files
      // pactUrls: ['./pacts/OrderService-UserService.json'],

      // Provider state handlers
      stateHandlers: {
        'user with ID 123 exists': async () => {
          await db.users.create({ id: '123', name: 'Alice', email: 'alice@test.com' });
        },
        'no users exist': async () => {
          await db.users.deleteAll();
        },
      },

      // Request filters (add auth headers, etc.)
      requestFilter: (req, res, next) => {
        req.headers.authorization = `Bearer ${generateTestToken()}`;
        next();
      },

      publishVerificationResult: process.env.CI === 'true',
      providerVersion: process.env.GIT_SHA,
      providerVersionBranch: process.env.GIT_BRANCH,
    });

    await verifier.verifyProvider();
  });
});
```

## Pact Broker Setup

```yaml
# docker-compose.yml — Pact Broker
services:
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports: ['9292:9292']
    environment:
      PACT_BROKER_DATABASE_URL: postgresql://pact:pact@postgres/pact
      PACT_BROKER_BASIC_AUTH_USERNAME: admin
      PACT_BROKER_BASIC_AUTH_PASSWORD: admin
    depends_on:
      postgres: { condition: service_healthy }

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
      POSTGRES_DB: pact
    healthcheck:
      test: pg_isready
      interval: 5s
```

## CI Integration for Contract Tests

```yaml
# Consumer pipeline
consumer-contract-test:
  steps:
    - run: npx vitest --project contract
    - name: Publish pacts
      run: |
        npx pact-broker publish ./pacts \
          --consumer-app-version=${{ github.sha }} \
          --branch=${{ github.ref_name }} \
          --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
          --broker-token=${{ secrets.PACT_BROKER_TOKEN }}
    - name: Can I deploy?
      run: |
        npx pact-broker can-i-deploy \
          --pacticipant=OrderService \
          --version=${{ github.sha }} \
          --to-environment=production

# Provider pipeline
provider-verification:
  steps:
    - run: npx vitest --project provider-verification
    - name: Record deployment
      if: github.ref == 'refs/heads/main'
      run: |
        npx pact-broker record-deployment \
          --pacticipant=UserService \
          --version=${{ github.sha }} \
          --environment=production
```

## Event-Driven Contract Testing

```typescript
// Message pact — async/event contracts
import { MessageConsumerPact, synchronousBodyHandler } from '@pact-foundation/pact';

// Consumer: expects events in a specific format
const messagePact = new MessageConsumerPact({
  consumer: 'NotificationService',
  provider: 'OrderService',
});

describe('Order events contract', () => {
  it('should handle order.created event', () => {
    return messagePact
      .given('an order is placed')
      .expectsToReceive('an order.created event')
      .withContent({
        eventType: 'order.created',
        data: {
          orderId: MatchersV3.uuid(),
          userId: MatchersV3.uuid(),
          total: MatchersV3.decimal(99.99),
          items: MatchersV3.eachLike({
            productId: MatchersV3.uuid(),
            quantity: MatchersV3.integer(1),
          }),
          createdAt: MatchersV3.iso8601DateTimeWithMillis(),
        },
      })
      .verify(
        synchronousBodyHandler(async (message) => {
          const handler = new OrderEventHandler();
          await handler.handle(message);
          // Assert handler processed correctly
        })
      );
  });
});

// Provider: verifies it produces correct events
const verifier = new Verifier({
  provider: 'OrderService',
  providerVersion: process.env.GIT_SHA,
  messageProviders: {
    'an order.created event': async () => {
      const order = await createTestOrder();
      return buildOrderCreatedEvent(order);
    },
  },
});
```

## GraphQL Contract Testing

```typescript
// GraphQL consumer contract
await provider
  .uponReceiving('a GraphQL query for user')
  .withRequest({
    method: 'POST',
    path: '/graphql',
    headers: { 'Content-Type': 'application/json' },
    body: {
      query: `query GetUser($id: ID!) {
        user(id: $id) { id name email }
      }`,
      variables: { id: '123' },
    },
  })
  .willRespondWith({
    status: 200,
    body: {
      data: {
        user: {
          id: MatchersV3.string('123'),
          name: MatchersV3.string('Alice'),
          email: MatchersV3.email(),
        },
      },
    },
  })
  .executeTest(async (mockServer) => {
    const client = new GraphQLClient(mockServer.url);
    const result = await client.getUser('123');
    expect(result.user.id).toBe('123');
  });
```

## Schema Evolution Strategy

```typescript
// Backward-compatible contract changes
// SAFE: Adding optional fields
// SAFE: Widening accepted types (string → string | number)
// UNSAFE: Removing fields
// UNSAFE: Changing field types
// UNSAFE: Making optional fields required

// Use Pact matchers for flexible contracts
MatchersV3.like({ name: 'default' })    // Match by type, not value
MatchersV3.eachLike({ id: '1' })        // Array with at least one item
MatchersV3.integer(42)                   // Any integer
MatchersV3.regex('\\d{4}-\\d{2}', '2025-01') // Pattern match
```
