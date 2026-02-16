# Node.js Security Hardening

## HTTP Security Headers (Helmet)

| Header | Purpose | Helmet Default |
|--------|---------|----------------|
| Content-Security-Policy | Prevent XSS, injection | Enabled |
| X-Content-Type-Options | Prevent MIME sniffing | nosniff |
| X-Frame-Options | Prevent clickjacking | SAMEORIGIN |
| Strict-Transport-Security | Force HTTPS | max-age=15552000 |
| X-DNS-Prefetch-Control | Control DNS prefetch | off |
| Referrer-Policy | Control referer header | no-referrer |

## CSRF Protection

```typescript
import { doubleCsrf } from 'csrf-csrf';

const { doubleCsrfProtection, generateToken } = doubleCsrf({
  getSecret: () => config.csrfSecret,
  cookieName: 'csrf-token',
  cookieOptions: { httpOnly: true, sameSite: 'strict', secure: true },
  getTokenFromRequest: (req) => req.headers['x-csrf-token'] as string,
});

app.use(doubleCsrfProtection);

// Expose CSRF token endpoint for SPA
app.get('/api/csrf-token', (req, res) => {
  res.json({ token: generateToken(req, res) });
});
```

## XSS Prevention

```typescript
import xss from 'xss';
import createDOMPurify from 'dompurify';
import { JSDOM } from 'jsdom';

// Sanitize HTML input
const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window as any);

function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'li'],
    ALLOWED_ATTR: ['href'],
  });
}

// Sanitize all string fields in request body
function sanitizeBody(req: Request, res: Response, next: NextFunction) {
  if (req.body && typeof req.body === 'object') {
    for (const [key, value] of Object.entries(req.body)) {
      if (typeof value === 'string') {
        req.body[key] = xss(value);
      }
    }
  }
  next();
}
```

## SQL Injection Prevention

```typescript
// ALWAYS use parameterized queries
// BAD
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`);

// GOOD — parameterized
const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);

// GOOD — Prisma (parameterized by default)
const user = await prisma.user.findUnique({ where: { email } });

// GOOD — Drizzle
const users = await db.select().from(usersTable).where(eq(usersTable.email, email));
```

## Request Size Limits

```typescript
app.use(express.json({ limit: '10kb' })); // Limit JSON body
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// File upload limits with multer
import multer from 'multer';
const upload = multer({
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  fileFilter: (req, file, cb) => {
    const allowed = ['image/jpeg', 'image/png', 'image/webp'];
    cb(null, allowed.includes(file.mimetype));
  },
});
```

## Dependency Auditing

```bash
# Check for vulnerabilities
npm audit
npm audit --audit-level=high

# Auto-fix where possible
npm audit fix

# CI enforcement
npm audit --audit-level=high || exit 1

# Lockfile integrity
npm ci  # Always use in CI
```

## Docker Security for Node.js

```dockerfile
# Use specific version, not latest
FROM node:20-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Install dependencies as root, then switch
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy app code
COPY --chown=appuser:appuser . .

# Switch to non-root
USER appuser

# Don't expose unnecessary info
ENV NODE_ENV=production

# Health check
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

## Secrets Management

```typescript
// Never hardcode secrets
// BAD: const secret = 'my-secret-key';

// Use environment variables validated at startup
const SecretsSchema = z.object({
  JWT_SECRET: z.string().min(32),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  API_KEY: z.string().min(16),
});
const secrets = SecretsSchema.parse(process.env);

// Never log secrets
function redactSecrets(obj: Record<string, unknown>): Record<string, unknown> {
  const sensitiveKeys = /secret|password|token|key|auth/i;
  return Object.fromEntries(
    Object.entries(obj).map(([k, v]) => [k, sensitiveKeys.test(k) ? '[REDACTED]' : v])
  );
}
```
