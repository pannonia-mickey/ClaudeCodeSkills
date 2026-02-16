---
name: Node.js Architecture
description: This skill should be used when the user asks about "Node.js project structure", "layered architecture Node", "NestJS modules", "dependency injection Node", "monorepo Node", "microservices Node", "Node.js clean architecture", or "event-driven Node". It covers project organization, architectural patterns, DI, and monorepo management.
---

# Node.js Architecture

## Layered Architecture

```
src/
  modules/
    users/
      users.controller.ts    # HTTP layer — routes, validation, response
      users.service.ts        # Business logic — rules, orchestration
      users.repository.ts     # Data access — database queries
      users.dto.ts            # Input/output schemas
      users.entity.ts         # Domain model
      users.module.ts         # Wiring (NestJS) or index
    orders/
      ...
  common/
    middleware/               # Auth, logging, error handling
    guards/                   # Authorization
    filters/                  # Exception filters
    interceptors/             # Response transformation
    decorators/               # Custom decorators
  config/
    database.config.ts
    app.config.ts
  app.ts                      # Application setup
  main.ts                     # Entry point
```

## Dependency Injection

```typescript
// Manual DI (no framework)
interface UserRepository {
  findById(id: string): Promise<User | null>;
  create(data: CreateUserInput): Promise<User>;
}

class UserService {
  constructor(private readonly repo: UserRepository) {}

  async getUser(id: string): Promise<User> {
    const user = await this.repo.findById(id);
    if (!user) throw new NotFoundError('User', id);
    return user;
  }
}

// Composition root
const prisma = new PrismaClient();
const userRepo = new PrismaUserRepository(prisma);
const userService = new UserService(userRepo);
const userController = new UserController(userService);
```

```typescript
// NestJS DI
@Injectable()
export class UserService {
  constructor(
    @Inject('USER_REPOSITORY') private readonly repo: UserRepository,
  ) {}
}

@Module({
  providers: [
    UserService,
    { provide: 'USER_REPOSITORY', useClass: PrismaUserRepository },
  ],
})
export class UsersModule {}
```

## Configuration Management

```typescript
import { z } from 'zod';

const ConfigSchema = z.object({
  port: z.coerce.number().default(3000),
  database: z.object({
    url: z.string().url(),
    poolSize: z.coerce.number().default(10),
  }),
  redis: z.object({
    url: z.string().url(),
  }),
  jwt: z.object({
    secret: z.string().min(32),
    expiresIn: z.string().default('15m'),
  }),
  logLevel: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export type Config = z.infer<typeof ConfigSchema>;

export const config: Config = ConfigSchema.parse({
  port: process.env.PORT,
  database: { url: process.env.DATABASE_URL, poolSize: process.env.DB_POOL_SIZE },
  redis: { url: process.env.REDIS_URL },
  jwt: { secret: process.env.JWT_SECRET, expiresIn: process.env.JWT_EXPIRES_IN },
  logLevel: process.env.LOG_LEVEL,
});
```

## Error Hierarchy

```typescript
abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;
  readonly isOperational = true;

  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  readonly statusCode = 404;
  readonly code = 'NOT_FOUND';
}

class UnauthorizedError extends AppError {
  readonly statusCode = 401;
  readonly code = 'UNAUTHORIZED';
}

class ForbiddenError extends AppError {
  readonly statusCode = 403;
  readonly code = 'FORBIDDEN';
}

class ConflictError extends AppError {
  readonly statusCode = 409;
  readonly code = 'CONFLICT';
}
```

## References

- [Architecture Patterns](references/architecture-patterns.md) — Clean Architecture, Hexagonal, event-driven, CQRS, microservices communication.
- [Monorepo Patterns](references/monorepo-patterns.md) — pnpm workspaces, Turborepo, Nx, shared packages, build strategies.
