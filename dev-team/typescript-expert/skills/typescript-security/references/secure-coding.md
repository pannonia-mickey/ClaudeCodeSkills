# TypeScript Secure Coding

## Type-Safe SQL (Parameterized Queries)

```typescript
// BAD — SQL injection vulnerable
const query = `SELECT * FROM users WHERE email = '${email}'`;

// GOOD — Parameterized with tagged template (e.g., slonik, postgres.js)
import postgres from 'postgres';
const sql = postgres();
const users = await sql`SELECT * FROM users WHERE email = ${email}`;

// GOOD — Query builder (Drizzle, Kysely)
const user = await db.select().from(users).where(eq(users.email, email));

// GOOD — Prisma (parameterized by default)
const user = await prisma.user.findUnique({ where: { email } });
```

## XSS Prevention

```typescript
// Sanitize HTML output
import DOMPurify from 'dompurify';

function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href', 'target'],
  });
}

// Use branded types to track sanitization
type SafeHtml = string & { readonly __brand: 'SafeHtml' };

function createSafeHtml(dirty: string): SafeHtml {
  return DOMPurify.sanitize(dirty) as SafeHtml;
}

function renderHtml(html: SafeHtml) { /* safe to use */ }
```

## Secrets Handling

```typescript
// Validate required secrets at startup
const SecretsSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  API_KEY: z.string().min(16),
  ENCRYPTION_KEY: z.string().length(64), // 256-bit hex
});

// Fail fast if secrets are missing
const secrets = SecretsSchema.parse(process.env);

// Never log secrets
function safeLog(data: Record<string, unknown>): void {
  const REDACTED_KEYS = ['password', 'secret', 'token', 'key', 'authorization'];
  const safe = Object.fromEntries(
    Object.entries(data).map(([k, v]) =>
      REDACTED_KEYS.some(rk => k.toLowerCase().includes(rk))
        ? [k, '[REDACTED]']
        : [k, v]
    )
  );
  console.log(safe);
}
```

## Dependency Auditing

```bash
# Check for known vulnerabilities
npm audit
npm audit --fix

# Automated in CI
npm audit --audit-level=high

# Use lockfile integrity checks
npm ci  # Always use in CI (respects lockfile exactly)

# Check for outdated packages
npm outdated

# Socket.dev for supply chain security
npx socket npm audit
```

## Type-Safe Fetch Wrapper

```typescript
async function safeFetch<T>(
  url: string,
  schema: z.ZodType<T>,
  init?: RequestInit,
): Promise<T> {
  const response = await fetch(url, init);

  if (!response.ok) {
    throw new HttpError(response.status, await response.text());
  }

  const data = await response.json();
  return schema.parse(data); // Validate response shape
}

class HttpError extends Error {
  constructor(
    public readonly status: number,
    public readonly body: string,
  ) {
    super(`HTTP ${status}: ${body}`);
    this.name = 'HttpError';
  }
}
```

## Avoiding Prototype Pollution

```typescript
// BAD — susceptible to prototype pollution
function merge(target: any, source: any) {
  for (const key in source) {
    target[key] = source[key]; // __proto__ injection!
  }
}

// GOOD — check for dangerous keys
function safeMerge<T extends Record<string, unknown>>(
  target: T,
  source: Partial<T>,
): T {
  for (const key of Object.keys(source)) {
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') continue;
    if (Object.hasOwn(source, key)) {
      (target as any)[key] = source[key as keyof T];
    }
  }
  return target;
}

// BEST — use Object.assign or structuredClone
const merged = { ...target, ...source };
const deepCopy = structuredClone(original);
```
