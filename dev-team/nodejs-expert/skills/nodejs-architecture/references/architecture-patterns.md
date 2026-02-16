# Node.js Architecture Patterns

## Clean Architecture (Ports & Adapters)

```
src/
  domain/              # Enterprise business rules (no dependencies)
    entities/
    value-objects/
    errors/
  application/         # Application business rules (depends on domain only)
    ports/             # Interfaces (inbound + outbound)
    use-cases/         # Application services
  infrastructure/      # Frameworks, drivers, external (depends on application)
    persistence/       # Database implementations
    http/              # Express/NestJS controllers
    messaging/         # Queue implementations
  main.ts              # Composition root — wires everything
```

```typescript
// Domain — pure, no dependencies
class Order {
  constructor(
    public readonly id: string,
    public readonly items: OrderItem[],
    public status: OrderStatus,
  ) {}

  get total(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  cancel(): void {
    if (this.status !== 'pending') throw new DomainError('Can only cancel pending orders');
    this.status = 'cancelled';
  }
}

// Application — use case orchestration
interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

interface NotificationService {
  notify(userId: string, message: string): Promise<void>;
}

class CancelOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly notifications: NotificationService,
  ) {}

  async execute(orderId: string): Promise<void> {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new NotFoundError('Order');
    order.cancel(); // Domain logic
    await this.orderRepo.save(order);
    await this.notifications.notify(order.userId, 'Your order was cancelled');
  }
}
```

## Event-Driven Patterns

```typescript
import { EventEmitter } from 'node:events';

// Typed event emitter
interface AppEvents {
  'user.created': [user: User];
  'order.placed': [order: Order];
  'order.cancelled': [orderId: string, reason: string];
}

class TypedEventBus extends EventEmitter {
  emit<K extends keyof AppEvents>(event: K, ...args: AppEvents[K]): boolean {
    return super.emit(event, ...args);
  }
  on<K extends keyof AppEvents>(event: K, listener: (...args: AppEvents[K]) => void): this {
    return super.on(event, listener);
  }
}

const eventBus = new TypedEventBus();
eventBus.on('user.created', (user) => sendWelcomeEmail(user));
eventBus.on('order.placed', (order) => updateInventory(order));
```

### Message Queue with BullMQ

```typescript
import { Queue, Worker } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis(process.env.REDIS_URL);

// Producer
const emailQueue = new Queue('emails', { connection });

await emailQueue.add('welcome', { userId: '123', template: 'welcome' }, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
});

// Consumer
const worker = new Worker('emails', async (job) => {
  const { userId, template } = job.data;
  await emailService.send(userId, template);
}, { connection, concurrency: 5 });

worker.on('completed', (job) => console.log(`Job ${job.id} completed`));
worker.on('failed', (job, err) => console.error(`Job ${job?.id} failed:`, err));
```

## Circuit Breaker

```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(
  async (userId: string) => {
    const response = await fetch(`https://external-api.com/users/${userId}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  },
  {
    timeout: 3000,           // Trip after 3s
    errorThresholdPercentage: 50, // Trip after 50% failure rate
    resetTimeout: 30000,     // Try again after 30s
    volumeThreshold: 5,      // Minimum calls before tripping
  }
);

breaker.fallback(() => ({ name: 'Unknown', cached: true }));
breaker.on('open', () => console.warn('Circuit opened'));
breaker.on('halfOpen', () => console.info('Circuit half-open'));
breaker.on('close', () => console.info('Circuit closed'));

const user = await breaker.fire('user-123');
```

## CQRS Pattern

```typescript
// Command — write operations
interface Command { type: string; }
interface CommandHandler<C extends Command> { execute(command: C): Promise<void>; }

class CreateOrderCommand implements Command {
  readonly type = 'CreateOrder';
  constructor(public readonly userId: string, public readonly items: OrderItem[]) {}
}

class CreateOrderHandler implements CommandHandler<CreateOrderCommand> {
  constructor(private repo: OrderRepository, private eventBus: EventBus) {}

  async execute(command: CreateOrderCommand): Promise<void> {
    const order = Order.create(command.userId, command.items);
    await this.repo.save(order);
    this.eventBus.emit('order.placed', order);
  }
}

// Query — read operations (can use denormalized read models)
class GetOrdersByUserQuery {
  constructor(public readonly userId: string) {}
}

class GetOrdersByUserHandler {
  constructor(private readDb: ReadOnlyDatabase) {}
  async execute(query: GetOrdersByUserQuery): Promise<OrderSummary[]> {
    return this.readDb.query('SELECT * FROM order_summaries WHERE user_id = $1', [query.userId]);
  }
}
```
