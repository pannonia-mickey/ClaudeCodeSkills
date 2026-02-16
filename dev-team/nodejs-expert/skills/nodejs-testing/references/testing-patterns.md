# Node.js Testing Patterns

## Testing NestJS Controllers

```typescript
describe('UsersController', () => {
  let controller: UsersController;
  let service: { findAll: ReturnType<typeof jest.fn>; create: ReturnType<typeof jest.fn> };

  beforeEach(async () => {
    service = { findAll: jest.fn(), create: jest.fn() };
    const module = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [{ provide: UsersService, useValue: service }],
    }).compile();
    controller = module.get(UsersController);
  });

  it('should return users', async () => {
    service.findAll.mockResolvedValue([{ id: '1', name: 'Alice' }]);
    const result = await controller.findAll({ page: 1, limit: 10 });
    expect(result).toHaveLength(1);
  });
});
```

## Testing Prisma Repositories

```typescript
import { mockDeep, DeepMockProxy } from 'jest-mock-extended';
import { PrismaClient } from '@prisma/client';

describe('UserRepository', () => {
  let prisma: DeepMockProxy<PrismaClient>;
  let repo: UserRepository;

  beforeEach(() => {
    prisma = mockDeep<PrismaClient>();
    repo = new UserRepository(prisma);
  });

  it('should find user by id', async () => {
    const mockUser = { id: '1', name: 'Alice', email: 'alice@test.com' };
    prisma.user.findUnique.mockResolvedValue(mockUser);

    const user = await repo.findById('1');

    expect(user).toEqual(mockUser);
    expect(prisma.user.findUnique).toHaveBeenCalledWith({ where: { id: '1' } });
  });

  it('should create user', async () => {
    const input = { name: 'Bob', email: 'bob@test.com' };
    prisma.user.create.mockResolvedValue({ id: '2', ...input });

    const user = await repo.create(input);

    expect(user.id).toBe('2');
    expect(prisma.user.create).toHaveBeenCalledWith({ data: input });
  });
});
```

## Testing WebSocket Handlers

```typescript
import { io as Client, Socket } from 'socket.io-client';
import { createServer } from '../src/server';

describe('WebSocket', () => {
  let server: ReturnType<typeof createServer>;
  let clientSocket: Socket;

  beforeAll((done) => {
    server = createServer();
    server.listen(0, () => {
      const port = (server.address() as any).port;
      clientSocket = Client(`http://localhost:${port}`, {
        auth: { token: testToken },
      });
      clientSocket.on('connect', done);
    });
  });

  afterAll(() => {
    clientSocket.close();
    server.close();
  });

  it('should receive notification', (done) => {
    clientSocket.on('notification', (data) => {
      expect(data.message).toBe('Hello');
      done();
    });
    clientSocket.emit('subscribe', { channel: 'test' });
  });
});
```

## Testing Background Jobs

```typescript
import { Queue, Worker } from 'bullmq';

describe('Email Worker', () => {
  let queue: Queue;

  beforeAll(() => {
    queue = new Queue('test-emails', { connection: testRedis });
  });

  afterAll(async () => {
    await queue.close();
  });

  it('should process email job', async () => {
    const sendEmailSpy = vi.spyOn(emailService, 'send').mockResolvedValue(undefined);

    const job = await queue.add('welcome', { userId: '1', template: 'welcome' });

    // Wait for processing
    await new Promise<void>((resolve) => {
      const worker = new Worker('test-emails', emailHandler, { connection: testRedis });
      worker.on('completed', async () => {
        expect(sendEmailSpy).toHaveBeenCalledWith('1', 'welcome');
        await worker.close();
        resolve();
      });
    });
  });
});
```

## Snapshot Testing API Responses

```typescript
it('should return correct response shape', async () => {
  const response = await request(app).get('/api/users/1').expect(200);

  expect(response.body).toMatchSnapshot({
    data: {
      id: expect.any(String),
      createdAt: expect.any(String),
      updatedAt: expect.any(String),
    },
  });
});
```
