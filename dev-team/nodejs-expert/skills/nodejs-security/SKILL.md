---
name: Node.js Security
description: This skill should be used when the user asks about "Node.js security", "Helmet", "Express rate limiting", "CSRF Node", "XSS prevention Node", "JWT best practices", "input sanitization Node", "dependency auditing", or "secrets management Node". It covers HTTP security headers, rate limiting, authentication, input validation, and dependency security.
---

# Node.js Security

## Helmet (HTTP Headers)

```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-abc123'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));
```

## Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';

// Global limit
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
  message: { error: { code: 'RATE_LIMITED', message: 'Too many requests' } },
}));

// Strict limit for auth
app.use('/api/auth', rateLimit({ windowMs: 15 * 60 * 1000, max: 5 }));
```

## JWT Authentication

```typescript
import jwt from 'jsonwebtoken';

// Token generation
function generateTokens(user: User) {
  const accessToken = jwt.sign(
    { sub: user.id, role: user.role },
    config.jwt.secret,
    { expiresIn: '15m' },
  );
  const refreshToken = jwt.sign(
    { sub: user.id, type: 'refresh' },
    config.jwt.refreshSecret,
    { expiresIn: '7d' },
  );
  return { accessToken, refreshToken };
}

// Auth middleware
function authenticate(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing token' });
  }
  try {
    const payload = jwt.verify(header.slice(7), config.jwt.secret);
    req.user = payload as TokenPayload;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Authorization
function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user!.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}
```

## Input Validation

```typescript
import { z } from 'zod';
import xss from 'xss';

// Sanitize + validate
const CreateUserSchema = z.object({
  email: z.string().email().max(254),
  name: z.string().min(1).max(100).transform(v => xss(v.trim())),
  password: z.string().min(8).max(72), // bcrypt max
});
```

## CORS Configuration

```typescript
import cors from 'cors';

app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? ['http://localhost:3000'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400,
}));
```

## References

- [Authentication Patterns](references/authentication-patterns.md) — JWT flow, Passport.js strategies, OAuth2/OIDC, session management, RBAC.
- [Hardening Guide](references/hardening-guide.md) — HTTP headers, CSRF, XSS, SQL injection, dependency auditing, Docker security.
