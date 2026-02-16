# TypeScript Advanced Patterns

## Branded Types

```typescript
// Prevent mixing semantically different values of the same base type
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;

function createUserId(id: string): UserId { return id as UserId; }
function createOrderId(id: string): OrderId { return id as OrderId; }

function getUser(id: UserId) { /* ... */ }

const userId = createUserId('abc');
const orderId = createOrderId('xyz');
getUser(userId);  // OK
// getUser(orderId); // Error: OrderId is not assignable to UserId
```

## Builder Pattern with Type Tracking

```typescript
type RequiredFields = 'host' | 'port';

class ServerBuilder<Built extends string = never> {
  private config: Record<string, unknown> = {};

  host(h: string): ServerBuilder<Built | 'host'> {
    this.config.host = h;
    return this as any;
  }

  port(p: number): ServerBuilder<Built | 'port'> {
    this.config.port = p;
    return this as any;
  }

  tls(enabled: boolean): ServerBuilder<Built | 'tls'> {
    this.config.tls = enabled;
    return this as any;
  }

  // build() only available when all required fields are set
  build(this: ServerBuilder<RequiredFields>): ServerConfig {
    return this.config as ServerConfig;
  }
}

new ServerBuilder().host('localhost').port(3000).build(); // OK
// new ServerBuilder().host('localhost').build(); // Error: missing 'port'
```

## Exhaustive Checking

```typescript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2;
    case 'rectangle': return shape.width * shape.height;
    case 'triangle': return (shape.base * shape.height) / 2;
    default: {
      const _exhaustive: never = shape; // Error if a case is missing
      throw new Error(`Unhandled shape: ${_exhaustive}`);
    }
  }
}
```

## Type-Safe Error Handling (Result Pattern)

```typescript
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

function ok<T>(data: T): Result<T, never> {
  return { success: true, data };
}

function err<E>(error: E): Result<never, E> {
  return { success: false, error };
}

// Usage
async function fetchUser(id: string): Promise<Result<User, 'not_found' | 'network_error'>> {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (res.status === 404) return err('not_found');
    return ok(await res.json());
  } catch {
    return err('network_error');
  }
}

const result = await fetchUser('123');
if (result.success) {
  console.log(result.data.name); // User
} else {
  console.error(result.error); // 'not_found' | 'network_error'
}
```

## Opaque Types with Symbols

```typescript
declare const __brand: unique symbol;

type Opaque<T, Token> = T & { readonly [__brand]: Token };

type Email = Opaque<string, 'Email'>;
type URL = Opaque<string, 'URL'>;

function validateEmail(input: string): Email | null {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input) ? (input as Email) : null;
}
```

## Type-Safe Event System

```typescript
type EventMap = {
  'user:login': { userId: string; timestamp: Date };
  'user:logout': { userId: string };
  'cart:update': { items: CartItem[]; total: number };
};

class TypedEmitter<Events extends Record<string, unknown>> {
  private listeners = new Map<keyof Events, Set<Function>>();

  on<K extends keyof Events>(event: K, handler: (payload: Events[K]) => void): void {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(handler);
  }

  emit<K extends keyof Events>(event: K, payload: Events[K]): void {
    this.listeners.get(event)?.forEach(handler => handler(payload));
  }
}

const emitter = new TypedEmitter<EventMap>();
emitter.on('user:login', ({ userId, timestamp }) => { /* fully typed */ });
// emitter.emit('user:login', { wrong: true }); // Error
```
