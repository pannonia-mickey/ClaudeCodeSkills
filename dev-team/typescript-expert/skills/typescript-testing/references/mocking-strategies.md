# TypeScript Mocking Strategies

## Module Mocking (Vitest)

```typescript
// Full module mock
vi.mock('@/services/database', () => ({
  getConnection: vi.fn().mockReturnValue({
    query: vi.fn(),
    close: vi.fn(),
  }),
}));

// Partial mock (keep original, override specific exports)
vi.mock('@/utils/helpers', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@/utils/helpers')>();
  return {
    ...actual,
    generateId: vi.fn().mockReturnValue('mock-id'),
  };
});

// Mock with factory (auto-mock all exports)
vi.mock('@/services/email'); // All exports become vi.fn()
```

## Class Mocking

```typescript
// Mock class with vi.fn()
class MockUserRepository implements UserRepository {
  findById = vi.fn<[string], Promise<User | null>>();
  create = vi.fn<[CreateUserInput], Promise<User>>();
  update = vi.fn<[string, Partial<User>], Promise<User>>();
  delete = vi.fn<[string], Promise<void>>();
}

// Auto-mock from interface
function createMockRepository<T extends Record<string, Function>>(): {
  [K in keyof T]: ReturnType<typeof vi.fn>;
} {
  return new Proxy({} as any, {
    get: (target, prop) => {
      if (!target[prop]) target[prop] = vi.fn();
      return target[prop];
    },
  });
}

const mockRepo = createMockRepository<UserRepository>();
```

## Dependency Injection for Testing

```typescript
// Production: real dependencies
class UserService {
  constructor(
    private readonly repo: UserRepository,
    private readonly emailService: EmailService,
    private readonly logger: Logger,
  ) {}

  async createUser(input: CreateUserInput): Promise<User> {
    const user = await this.repo.create(input);
    await this.emailService.sendWelcome(user.email);
    this.logger.info('User created', { userId: user.id });
    return user;
  }
}

// Test: mock dependencies
describe('UserService', () => {
  let service: UserService;
  let mockRepo: MockUserRepository;
  let mockEmail: { sendWelcome: ReturnType<typeof vi.fn> };
  let mockLogger: { info: ReturnType<typeof vi.fn>; error: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    mockRepo = new MockUserRepository();
    mockEmail = { sendWelcome: vi.fn() };
    mockLogger = { info: vi.fn(), error: vi.fn() };
    service = new UserService(mockRepo, mockEmail as any, mockLogger as any);
  });

  it('should create user and send welcome email', async () => {
    const user: User = { id: '1', name: 'Alice', email: 'alice@test.com' };
    mockRepo.create.mockResolvedValue(user);

    await service.createUser({ name: 'Alice', email: 'alice@test.com' });

    expect(mockRepo.create).toHaveBeenCalledOnce();
    expect(mockEmail.sendWelcome).toHaveBeenCalledWith('alice@test.com');
    expect(mockLogger.info).toHaveBeenCalledWith('User created', { userId: '1' });
  });
});
```

## MSW (Mock Service Worker) for HTTP

```typescript
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: 'Alice',
      email: 'alice@example.com',
    });
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 'new-id', ...body }, { status: 201 });
  }),

  http.get('/api/users', ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page') ?? 1);
    return HttpResponse.json({
      data: [{ id: '1', name: 'Alice' }],
      meta: { page, total: 1 },
    });
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Override for specific test
it('should handle server error', async () => {
  server.use(
    http.get('/api/users/:id', () => {
      return new HttpResponse(null, { status: 500 });
    }),
  );

  await expect(fetchUser('1')).rejects.toThrow();
});
```

## Spy Patterns

```typescript
// Spy on object method
const logger = new Logger();
const spy = vi.spyOn(logger, 'info');

await doSomething(logger);
expect(spy).toHaveBeenCalledWith('Operation complete', expect.objectContaining({ duration: expect.any(Number) }));

spy.mockRestore(); // Restore original implementation

// Spy on module function
import * as utils from '@/utils';
const spy = vi.spyOn(utils, 'generateId').mockReturnValue('fixed-id');

// Spy on global
const consoleSpy = vi.spyOn(console, 'error').mockImplementation(() => {});
```

## Testing Error Paths

```typescript
it('should handle repository errors gracefully', async () => {
  mockRepo.findById.mockRejectedValue(new DatabaseError('Connection lost'));

  const result = await service.getUser('1');

  expect(result).toEqual({ success: false, error: 'Service unavailable' });
  expect(mockLogger.error).toHaveBeenCalledWith(
    'Failed to fetch user',
    expect.objectContaining({ error: expect.any(DatabaseError) }),
  );
});

it('should propagate validation errors', async () => {
  await expect(service.createUser({ email: 'invalid' })).rejects.toThrow(ValidationError);
});
```
