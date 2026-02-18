# Practice Patterns Reference

Current standards vs. outdated patterns across major development categories. Use this as a quick-reference for identifying whether a found approach reflects current best practice.

---

## API Design

### REST API Design

| Pattern | Status | Notes |
|---------|--------|-------|
| Resource-based URLs (`/users/{id}`) | ✓ Current standard | Noun, not verb |
| HTTP verbs for CRUD (GET/POST/PUT/PATCH/DELETE) | ✓ Current standard | |
| Version in URL (`/api/v1/`) | ✓ Acceptable | Or via Accept-Version header |
| Nested resources beyond 2 levels (`/users/1/orders/2/items`) | ⚠ Avoid | Prefer `/items?userId=1&orderId=2` |
| Returning raw arrays at root level (`[{...}]`) | ⚠ Avoid | Wrap in object for extensibility |
| Exposing database IDs as sequential integers | ⚠ Avoid | Use UUIDs or opaque identifiers |
| Verb-based URLs (`/getUser`, `/createUser`) | ✗ Outdated | Use HTTP methods on resource URLs |
| Returning errors with HTTP 200 | ✗ Outdated | Use appropriate HTTP status codes |

### GraphQL

| Pattern | Status | Notes |
|---------|--------|-------|
| Code-first schema with resolvers | ✓ Current | Type-safe with TypeScript |
| Persisted queries | ✓ Best practice | Performance + security |
| DataLoader for batching | ✓ Essential | Prevents N+1 problem |
| Blocking nested queries without depth limiting | ✗ Security risk | Always implement query depth/complexity limits |
| Exposing raw database models as GraphQL types | ⚠ Avoid | Use separate DTO/view models |

---

## Authentication

| Pattern | Status | Notes |
|---------|--------|-------|
| JWT with short expiry + refresh token rotation | ✓ Current standard | Access tokens < 15min |
| HttpOnly + Secure + SameSite cookies for session tokens | ✓ Current standard | Prevents XSS token theft |
| OAuth 2.0 + PKCE for third-party auth | ✓ Current standard | PKCE required for SPAs |
| Passkeys / WebAuthn | ✓ Emerging best practice | Replacing passwords |
| bcrypt or Argon2 for password hashing | ✓ Current standard | Work factor ≥ 12 |
| Auth.js (formerly NextAuth) for Next.js | ✓ Current recommendation | |
| Storing JWT in localStorage | ✗ Security risk | XSS vulnerable — use HttpOnly cookies |
| MD5 or SHA-1 for password hashing | ✗ Insecure | Never use |
| Basic Auth over HTTP | ✗ Insecure | HTTPS required at minimum |
| Rolling your own auth from scratch | ⚠ High risk | Use established libraries |
| Passport.js for new Node.js projects | ⚠ Dated | Consider Auth.js or Lucia |

---

## Error Handling

### Node.js / JavaScript

| Pattern | Status | Notes |
|---------|--------|-------|
| Custom error classes extending Error | ✓ Current standard | Type-safe error handling |
| `instanceof` checks for error routing | ✓ Current standard | |
| Result type pattern (neverthrow, Effect) | ✓ Emerging | Explicit error types |
| Structured logging with error context | ✓ Current standard | |
| Catching all errors with empty catch block | ✗ Anti-pattern | Always handle or rethrow |
| `process.on('uncaughtException')` as only handler | ✗ Anti-pattern | Pair with `unhandledRejection` |
| `console.error` in production | ⚠ Insufficient | Use structured logger (pino, winston) |

### Python

| Pattern | Status | Notes |
|---------|--------|-------|
| Custom exception hierarchy from base Exception | ✓ Current standard | |
| `try/except SpecificError` | ✓ Current standard | Never bare `except:` |
| Context managers for resource cleanup | ✓ Current standard | `with` statement |
| Bare `except:` or `except Exception:` | ✗ Anti-pattern | Swallows unexpected errors |
| `raise Exception("message")` (base class) | ⚠ Avoid | Create specific exception types |

---

## Testing

### JavaScript / TypeScript

| Pattern | Status | Notes |
|---------|--------|-------|
| Vitest for unit tests | ✓ Current standard for Vite projects | Fast, Jest-compatible |
| Jest for unit tests | ✓ Still current | Dominant for non-Vite projects |
| React Testing Library | ✓ Current standard | Test behavior, not implementation |
| Playwright for E2E | ✓ Current standard | Cross-browser, async |
| Mock Service Worker (MSW) for API mocking | ✓ Current standard | Intercepts at network level |
| `describe/it/expect` structure | ✓ Current standard | |
| Enzyme for React component testing | ✗ Deprecated | RTL is the replacement |
| Testing implementation details (calling private methods) | ✗ Anti-pattern | Test public interface/behavior |
| Snapshot testing for everything | ⚠ Overused | Useful for output rendering, not logic |
| 100% code coverage as goal | ⚠ Misleading | Focus on critical paths and edge cases |

### Python

| Pattern | Status | Notes |
|---------|--------|-------|
| pytest | ✓ Current standard | Over unittest for new projects |
| pytest-asyncio for async tests | ✓ Current standard | |
| httpx + respx for HTTP client testing | ✓ Current standard | |
| Factory Boy for test data | ✓ Current standard | Over manual fixtures for complex models |
| unittest | ⚠ Viable but dated | pytest is preferred for new code |
| Mocking with `unittest.mock` | ✓ Current standard | Or pytest-mock for cleaner syntax |

---

## CI/CD

| Pattern | Status | Notes |
|---------|--------|-------|
| GitHub Actions for CI | ✓ Current standard | Dominant for GitHub-hosted repos |
| Automated tests on every PR | ✓ Essential | Gate merges on passing tests |
| Semantic versioning for releases | ✓ Current standard | |
| Trunk-based development | ✓ Best practice | Short-lived feature branches |
| Container-based CI (Docker in CI) | ✓ Current standard | Consistent environments |
| Dependabot or Renovate for dependency updates | ✓ Best practice | Automated security patches |
| Manual FTP deployment | ✗ Outdated | Use any CI/CD pipeline |
| Long-lived feature branches (weeks+) | ⚠ Risk | Increases merge conflicts |
| Hardcoding secrets in CI config | ✗ Security risk | Use CI secrets management |
| `npm install` without lockfile in CI | ✗ Anti-pattern | Use `npm ci` |

---

## Security

| Pattern | Status | Notes |
|---------|--------|-------|
| HTTPS everywhere | ✓ Essential | Let's Encrypt makes it free |
| CSP (Content Security Policy) headers | ✓ Current standard | Prevents XSS |
| Input validation at API boundary | ✓ Essential | Never trust client data |
| Parameterized queries / ORM | ✓ Essential | Prevents SQL injection |
| Secrets in environment variables | ✓ Current standard | Never in code |
| Secrets management service (Vault, AWS Secrets Manager) | ✓ Best practice for production | |
| Rate limiting on auth endpoints | ✓ Essential | |
| CORS properly configured | ✓ Essential | Not `*` in production |
| eval() on user input | ✗ Never | Code injection |
| SQL string interpolation | ✗ Never | SQL injection |
| Secrets in git history | ✗ Never | Rotate immediately if committed |
| `X-Powered-By: Express` header | ⚠ Remove | Information disclosure |
| Disabled HTTPS in production | ✗ Never | |

---

## State Management (JavaScript/React)

| Pattern | Status | Notes |
|---------|--------|-------|
| useState + useReducer for local state | ✓ Current standard | Component-level |
| TanStack Query for server/async state | ✓ Current standard | Replaces manual fetch + state |
| Zustand for simple global state | ✓ Current standard | Small API, no boilerplate |
| Redux Toolkit for complex global state | ✓ Current standard (when Redux needed) | Not Redux core alone |
| Jotai / Recoil for atomic state | ✓ Viable | Fine-grained reactivity use case |
| Context API for frequently-changing global state | ⚠ Performance risk | Causes excess re-renders |
| Redux without Redux Toolkit | ⚠ Dated | Unnecessary boilerplate |
| MobX | ⚠ Viable but declining | Good if already in codebase |
| Flux pattern implementations | ✗ Superseded | Redux/Zustand are replacements |

---

## Data Access / ORM

| Pattern | Status | Notes |
|---------|--------|-------|
| Prisma for Node.js + TypeScript | ✓ Current standard | Type-safe, good DX |
| SQLAlchemy 2.x for Python | ✓ Current standard | Async support, typed |
| TypeORM | ⚠ Viable but issues | Prisma preferred for new TS projects |
| Raw SQL with pg/mysql2 (Node) | ✓ Valid for complex queries | Use alongside ORM |
| Drizzle ORM | ✓ Emerging | Lightweight, SQL-first |
| Sequelize | ⚠ Dated | TypeORM/Prisma preferred |
| Active Record pattern in non-Rails contexts | ⚠ Context-dependent | Repository pattern often cleaner |
