---
name: Node.js Testing
description: This skill should be used when the user asks about "test Express API", "test NestJS", "supertest", "nock HTTP mock", "Vitest Node", "Jest Node", "Testcontainers Node", "E2E test Node API", or "mock database Node". It covers unit testing, HTTP testing, mocking, and integration testing for Node.js applications.
---

# Node.js Testing

## Unit Testing Services

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';

describe('UserService', () => {
  let service: UserService;
  let mockRepo: { findById: ReturnType<typeof vi.fn>; create: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    mockRepo = { findById: vi.fn(), create: vi.fn() };
    service = new UserService(mockRepo);
  });

  it('should return user by id', async () => {
    mockRepo.findById.mockResolvedValue({ id: '1', name: 'Alice' });
    const user = await service.getUser('1');
    expect(user).toEqual({ id: '1', name: 'Alice' });
    expect(mockRepo.findById).toHaveBeenCalledWith('1');
  });

  it('should throw NotFound when user missing', async () => {
    mockRepo.findById.mockResolvedValue(null);
    await expect(service.getUser('999')).rejects.toThrow(NotFoundError);
  });
});
```

## HTTP Testing with Supertest

```typescript
import request from 'supertest';
import { app } from '../src/app';

describe('POST /api/users', () => {
  it('should create a user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'alice@example.com' })
      .expect(201);

    expect(response.body.data).toMatchObject({
      name: 'Alice',
      email: 'alice@example.com',
    });
    expect(response.body.data.id).toBeDefined();
  });

  it('should reject invalid email', async () => {
    await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'not-an-email' })
      .expect(400);
  });

  it('should require authentication', async () => {
    await request(app)
      .get('/api/users/me')
      .expect(401);
  });

  it('should work with auth token', async () => {
    const token = generateTestToken({ sub: '1', role: 'user' });
    await request(app)
      .get('/api/users/me')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
  });
});
```

## Mocking HTTP with nock

```typescript
import nock from 'nock';

describe('ExternalApiService', () => {
  afterEach(() => nock.cleanAll());

  it('should fetch external data', async () => {
    nock('https://api.external.com')
      .get('/data')
      .reply(200, { items: [{ id: 1 }] });

    const result = await service.fetchExternalData();
    expect(result.items).toHaveLength(1);
  });

  it('should handle external API errors', async () => {
    nock('https://api.external.com')
      .get('/data')
      .reply(500, { error: 'Internal Server Error' });

    await expect(service.fetchExternalData()).rejects.toThrow();
  });
});
```

## Testing Middleware

```typescript
import { Request, Response, NextFunction } from 'express';

describe('authenticate middleware', () => {
  let req: Partial<Request>;
  let res: Partial<Response>;
  let next: NextFunction;

  beforeEach(() => {
    req = { headers: {} };
    res = { status: vi.fn().mockReturnThis(), json: vi.fn() };
    next = vi.fn();
  });

  it('should call next with valid token', () => {
    req.headers = { authorization: `Bearer ${validToken}` };
    authenticate(req as Request, res as Response, next);
    expect(next).toHaveBeenCalled();
    expect((req as any).user).toBeDefined();
  });

  it('should return 401 without token', () => {
    authenticate(req as Request, res as Response, next);
    expect(res.status).toHaveBeenCalledWith(401);
    expect(next).not.toHaveBeenCalled();
  });
});
```

## References

- [Testing Patterns](references/testing-patterns.md) — Service testing, repository testing, middleware testing, guard testing.
- [Integration Testing](references/integration-testing.md) — Testcontainers, database fixtures, API contract testing, CI setup.
