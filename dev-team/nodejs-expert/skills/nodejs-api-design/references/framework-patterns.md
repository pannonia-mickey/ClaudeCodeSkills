# Node.js Framework Patterns

## Framework Comparison

| Feature | Express | NestJS | Hono | Fastify |
|---------|---------|--------|------|---------|
| Architecture | Middleware chain | Module/DI | Middleware chain | Plugin system |
| TypeScript | Manual setup | Built-in | Built-in | Built-in |
| Validation | Third-party | class-validator | Zod middleware | JSON Schema |
| Performance | Good | Good | Excellent | Excellent |
| Best For | Flexibility | Enterprise apps | Edge/serverless | High perf APIs |
| Learning Curve | Low | High | Low | Medium |

## Express Middleware Chain

```typescript
// Application-level middleware
app.use(cors());
app.use(helmet());
app.use(express.json({ limit: '10kb' }));
app.use(requestLogger); // Custom logging

// Router-level middleware
const router = express.Router();
router.use(authenticate); // All routes in this router require auth

// Route-specific middleware
router.get('/admin', authorize('admin'), adminController.dashboard);

// Error-handling middleware (4 args)
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: err.message });
  }
  logger.error('Unhandled error', { error: err, path: req.path });
  res.status(500).json({ error: 'Internal server error' });
});

// Async wrapper (avoids try/catch in every handler)
const asyncHandler = (fn: Function) => (req: Request, res: Response, next: NextFunction) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

router.get('/users', asyncHandler(async (req, res) => {
  const users = await userService.findAll();
  res.json({ data: users });
}));
```

## NestJS Module System

```typescript
// Module — organizes related providers, controllers, imports
@Module({
  imports: [PrismaModule, AuthModule],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService], // Available to other modules
})
export class UsersModule {}

// Guard — authentication/authorization
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);
    if (!token) throw new UnauthorizedException();
    request.user = await this.jwtService.verify(token);
    return true;
  }
}

// Interceptor — transform response, add logging
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    return next.handle().pipe(
      map(data => ({ data, timestamp: new Date().toISOString() })),
    );
  }
}

// Pipe — transform/validate input
@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodType) {}
  transform(value: unknown) {
    return this.schema.parse(value);
  }
}

// Exception filter — custom error responses
@Catch(PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  catch(exception: PrismaClientKnownRequestError, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    if (exception.code === 'P2025') {
      response.status(404).json({ error: 'Resource not found' });
    } else {
      response.status(500).json({ error: 'Database error' });
    }
  }
}
```

## Fastify Plugin System

```typescript
import Fastify from 'fastify';

const app = Fastify({ logger: true });

// Plugin (encapsulated scope)
async function usersPlugin(fastify: FastifyInstance) {
  fastify.get('/users', {
    schema: {
      querystring: { type: 'object', properties: { page: { type: 'integer' } } },
      response: { 200: { type: 'array', items: UserSchema } },
    },
  }, async (request, reply) => {
    return userService.findAll(request.query);
  });
}

app.register(usersPlugin, { prefix: '/api' });

// Decorators — extend Fastify instance
app.decorate('db', prisma);
app.decorateRequest('user', null);

// Hooks
app.addHook('onRequest', async (request) => {
  request.user = await authenticate(request.headers.authorization);
});
```

## Hono Middleware

```typescript
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { logger } from 'hono/logger';
import { jwt } from 'hono/jwt';

const app = new Hono();

app.use('*', logger());
app.use('*', cors({ origin: 'https://example.com' }));
app.use('/api/*', jwt({ secret: process.env.JWT_SECRET! }));

// Custom middleware
app.use('/api/*', async (c, next) => {
  const start = Date.now();
  await next();
  const duration = Date.now() - start;
  c.header('X-Response-Time', `${duration}ms`);
});

// Error handling
app.onError((err, c) => {
  if (err instanceof AppError) {
    return c.json({ error: err.message }, err.statusCode as any);
  }
  return c.json({ error: 'Internal server error' }, 500);
});
```
