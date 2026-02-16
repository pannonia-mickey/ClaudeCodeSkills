---
name: Node.js API Design
description: This skill should be used when the user asks to "create Express API", "NestJS controller", "Hono route", "Fastify plugin", "REST API Node", "Express middleware", "API validation", "Express error handling", or "OpenAPI Node". It covers Express, NestJS, Hono, and Fastify patterns for API design.
---

# Node.js API Design

## Express.js Pattern

```typescript
import express from 'express';
import { z } from 'zod';

const app = express();
app.use(express.json());

// Validation middleware factory
function validate<T>(schema: z.ZodType<T>) {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({ errors: result.error.flatten().fieldErrors });
    }
    req.body = result.data;
    next();
  };
}

// Router
const router = express.Router();

router.get('/', async (req, res, next) => {
  try {
    const users = await userService.findAll(req.query);
    res.json({ data: users });
  } catch (err) { next(err); }
});

router.post('/', validate(CreateUserSchema), async (req, res, next) => {
  try {
    const user = await userService.create(req.body);
    res.status(201).json({ data: user });
  } catch (err) { next(err); }
});

app.use('/api/users', router);

// Centralized error handler
app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: err.message, code: err.code });
  }
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
});
```

## NestJS Pattern

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll(@Query() query: PaginationDto) {
    return this.usersService.findAll(query);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @UsePipes(new ValidationPipe({ whitelist: true }))
  create(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }

  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string) {
    const user = await this.usersService.findOne(id);
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }
}
```

## Hono Pattern

```typescript
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';

const app = new Hono();

const users = new Hono()
  .get('/', async (c) => {
    const users = await userService.findAll();
    return c.json({ data: users });
  })
  .post('/', zValidator('json', CreateUserSchema), async (c) => {
    const data = c.req.valid('json');
    const user = await userService.create(data);
    return c.json({ data: user }, 201);
  })
  .get('/:id', async (c) => {
    const user = await userService.findOne(c.req.param('id'));
    if (!user) return c.json({ error: 'Not found' }, 404);
    return c.json({ data: user });
  });

app.route('/api/users', users);
```

## Error Hierarchy

```typescript
class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code: string,
  ) { super(message); this.name = 'AppError'; }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, 404, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(message: string, public readonly fields: Record<string, string[]>) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, 'CONFLICT');
  }
}
```

## References

- [Framework Patterns](references/framework-patterns.md) — Express vs NestJS vs Hono vs Fastify comparison, middleware chains, plugin systems.
- [API Best Practices](references/api-best-practices.md) — Pagination, filtering, versioning, rate limiting, OpenAPI generation.
