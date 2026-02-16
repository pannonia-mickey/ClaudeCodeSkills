---
name: Testing Mastery
description: This skill should be used when the user asks about "test pyramid", "AAA pattern", "test doubles", "test fixtures", "test isolation", "deterministic testing", "flaky tests", "CI test integration", or "testing fundamentals". It covers core testing concepts, patterns, and principles applicable across all frameworks.
---

# Testing Fundamentals

## Test Pyramid

```
          /  E2E  \          Few, slow, expensive
         /----------\
        / Integration \      Moderate number
       /----------------\
      /    Unit Tests     \  Many, fast, cheap
     /____________________\
```

- **Unit** (70%): Test individual functions/classes in isolation. Fast, deterministic.
- **Integration** (20%): Test component interactions, database queries, API calls.
- **E2E** (10%): Test complete user journeys through the real system.

## AAA Pattern (Arrange-Act-Assert)

```typescript
it('should calculate order total with discount', () => {
  // Arrange — set up test data and dependencies
  const order = createOrder({ items: [{ price: 100, quantity: 2 }] });
  const discount = { type: 'percentage', value: 10 };

  // Act — execute the behavior under test
  const total = calculateTotal(order, discount);

  // Assert — verify the expected outcome
  expect(total).toBe(180); // 200 - 10%
});
```

## Test Doubles

```typescript
// Stub — returns canned responses
const userRepo = { findById: vi.fn().mockResolvedValue({ id: '1', name: 'Alice' }) };

// Mock — verifies interactions
const emailService = { send: vi.fn() };
await service.createUser(data);
expect(emailService.send).toHaveBeenCalledWith('alice@test.com', 'Welcome!');

// Spy — wraps real implementation, records calls
const spy = vi.spyOn(logger, 'info');
await doWork();
expect(spy).toHaveBeenCalledWith('Work completed');

// Fake — working implementation with shortcuts
class InMemoryUserRepo implements UserRepository {
  private users = new Map<string, User>();
  async findById(id: string) { return this.users.get(id) ?? null; }
  async save(user: User) { this.users.set(user.id, user); }
}
```

## Test Isolation

```typescript
// Each test must be independent
describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    // Fresh instance for every test
    service = new UserService(new InMemoryUserRepo());
  });

  // Tests can run in any order, in parallel
  it('test A', async () => { /* ... */ });
  it('test B', async () => { /* ... */ });
});

// Database test isolation
afterEach(async () => {
  await db.execute('TRUNCATE users, orders CASCADE');
});
```

## Fixing Flaky Tests

Common causes and fixes:
- **Time-dependent**: Use fake timers, avoid `sleep()`
- **Order-dependent**: Reset state in beforeEach/afterEach
- **Race conditions**: Use proper async/await, avoid polling
- **External dependencies**: Mock external services
- **Shared state**: Isolate test databases, use transactions

## References

- [Test Patterns](references/test-patterns.md) — Test factories, builders, custom matchers, parameterized tests.
- [CI Integration](references/ci-integration.md) — Parallel execution, test splitting, reporting, artifact management.
