# Penetration Testing Patterns

## OWASP Top 10 Test Cases

### A01: Broken Access Control

```typescript
describe('Access control tests', () => {
  // Vertical privilege escalation
  it('should prevent regular user from accessing admin endpoints', async () => {
    const userToken = await loginAs('regular-user');
    const adminEndpoints = [
      '/api/admin/users',
      '/api/admin/settings',
      '/api/admin/audit-log',
    ];

    for (const endpoint of adminEndpoints) {
      const res = await request(app)
        .get(endpoint)
        .set('Authorization', `Bearer ${userToken}`);
      expect(res.status).toBe(403);
    }
  });

  // Horizontal privilege escalation
  it('should prevent accessing other user resources via IDOR', async () => {
    const user1 = await createAndLogin('user1@test.com');
    const user2 = await createAndLogin('user2@test.com');

    // User1 creates a resource
    const resource = await request(app)
      .post('/api/documents')
      .set('Authorization', `Bearer ${user1.token}`)
      .send({ title: 'Private doc', content: 'Secret' });

    // User2 tries to access it
    const res = await request(app)
      .get(`/api/documents/${resource.body.id}`)
      .set('Authorization', `Bearer ${user2.token}`);
    expect(res.status).toBe(403);
  });

  // Path traversal
  it('should prevent path traversal in file access', async () => {
    const payloads = [
      '../../../etc/passwd',
      '..\\..\\..\\windows\\system32\\config\\sam',
      '%2e%2e%2f%2e%2e%2f',
      '....//....//....//etc/passwd',
    ];

    for (const payload of payloads) {
      const res = await request(app).get(`/api/files/${payload}`);
      expect(res.status).toBeOneOf([400, 403, 404]);
      expect(res.text).not.toContain('root:');
    }
  });
});
```

### A02: Cryptographic Failures

```typescript
describe('Cryptographic security', () => {
  it('should not expose sensitive data in responses', async () => {
    const res = await request(app)
      .get('/api/users/me')
      .set('Authorization', `Bearer ${token}`);

    expect(res.body).not.toHaveProperty('password');
    expect(res.body).not.toHaveProperty('passwordHash');
    expect(res.body).not.toHaveProperty('ssn');
    expect(JSON.stringify(res.body)).not.toMatch(/\$2[aby]\$\d{2}\$/); // bcrypt hash
  });

  it('should use HTTPS redirect', async () => {
    const res = await request(app)
      .get('/')
      .set('X-Forwarded-Proto', 'http');
    expect(res.headers['strict-transport-security']).toBeDefined();
  });
});
```

### A03: Injection

```typescript
describe('Injection prevention', () => {
  const sqlPayloads = [
    "' OR '1'='1",
    "'; DROP TABLE users; --",
    "' UNION SELECT * FROM users --",
    "1; WAITFOR DELAY '0:0:5'--",
  ];

  const xssPayloads = [
    '<script>alert(1)</script>',
    '<img src=x onerror=alert(1)>',
    '"><svg onload=alert(1)>',
    "javascript:alert('XSS')",
  ];

  const noSqlPayloads = [
    '{"$gt": ""}',
    '{"$ne": null}',
    '{"$where": "sleep(5000)"}',
  ];

  for (const payload of sqlPayloads) {
    it(`should handle SQL injection: ${payload.slice(0, 30)}`, async () => {
      const res = await request(app)
        .get('/api/users')
        .query({ search: payload });
      expect(res.status).toBeOneOf([200, 400]);
      // Should return empty results or validation error, never a DB error
      expect(res.status).not.toBe(500);
    });
  }

  for (const payload of xssPayloads) {
    it(`should sanitize XSS: ${payload.slice(0, 30)}`, async () => {
      // Store payload
      await request(app)
        .post('/api/comments')
        .set('Authorization', `Bearer ${token}`)
        .send({ text: payload });

      // Retrieve and verify sanitized
      const res = await request(app).get('/api/comments');
      expect(res.text).not.toContain('<script');
      expect(res.text).not.toContain('onerror=');
      expect(res.text).not.toContain('onload=');
    });
  }
});
```

## API Security Testing

```typescript
describe('API security', () => {
  // Mass assignment / overposting
  it('should not allow mass assignment of admin role', async () => {
    const res = await request(app)
      .post('/api/auth/register')
      .send({
        email: 'attacker@test.com',
        password: 'Password123!',
        role: 'admin',          // Should be ignored
        isVerified: true,        // Should be ignored
      });

    expect(res.body.role).toBe('user');
    expect(res.body.isVerified).toBe(false);
  });

  // Excessive data exposure
  it('should not return internal fields in list endpoints', async () => {
    const res = await request(app)
      .get('/api/users')
      .set('Authorization', `Bearer ${adminToken}`);

    for (const user of res.body.data) {
      expect(user).not.toHaveProperty('passwordHash');
      expect(user).not.toHaveProperty('resetToken');
      expect(user).not.toHaveProperty('loginAttempts');
    }
  });

  // Request size limits
  it('should reject oversized payloads', async () => {
    const largePayload = { data: 'x'.repeat(10 * 1024 * 1024) }; // 10MB
    const res = await request(app)
      .post('/api/data')
      .set('Authorization', `Bearer ${token}`)
      .send(largePayload);
    expect(res.status).toBe(413);
  });

  // Content-type validation
  it('should reject unexpected content types', async () => {
    const res = await request(app)
      .post('/api/users')
      .set('Content-Type', 'text/xml')
      .send('<user><name>test</name></user>');
    expect(res.status).toBeOneOf([400, 415]);
  });
});
```

## Authentication Bypass Tests

```typescript
describe('Authentication bypass prevention', () => {
  // JWT manipulation
  it('should reject JWT with none algorithm', async () => {
    const header = Buffer.from('{"alg":"none","typ":"JWT"}').toString('base64url');
    const payload = Buffer.from('{"sub":"admin","role":"admin"}').toString('base64url');
    const fakeToken = `${header}.${payload}.`;

    const res = await request(app)
      .get('/api/admin')
      .set('Authorization', `Bearer ${fakeToken}`);
    expect(res.status).toBe(401);
  });

  // Token reuse after logout
  it('should invalidate token after logout', async () => {
    const { token } = await loginAs('user@test.com');

    // Logout
    await request(app)
      .post('/api/auth/logout')
      .set('Authorization', `Bearer ${token}`);

    // Try to use old token
    const res = await request(app)
      .get('/api/users/me')
      .set('Authorization', `Bearer ${token}`);
    expect(res.status).toBe(401);
  });

  // Brute force protection
  it('should lock account after failed attempts', async () => {
    for (let i = 0; i < 10; i++) {
      await request(app)
        .post('/api/auth/login')
        .send({ email: 'user@test.com', password: 'wrong' });
    }

    // Even with correct password
    const res = await request(app)
      .post('/api/auth/login')
      .send({ email: 'user@test.com', password: 'CorrectPassword123!' });
    expect(res.status).toBe(429);
    expect(res.body.message).toMatch(/locked|too many/i);
  });

  // Password reset token security
  it('should not reuse password reset tokens', async () => {
    // Request reset
    await request(app)
      .post('/api/auth/forgot-password')
      .send({ email: 'user@test.com' });

    const resetToken = await getLatestResetToken('user@test.com');

    // Use token
    await request(app)
      .post('/api/auth/reset-password')
      .send({ token: resetToken, newPassword: 'NewPassword123!' });

    // Try to reuse
    const res = await request(app)
      .post('/api/auth/reset-password')
      .send({ token: resetToken, newPassword: 'AnotherPassword123!' });
    expect(res.status).toBeOneOf([400, 401]);
  });
});
```

## Security Test Organization

```typescript
// vitest.config.security.ts
export default defineConfig({
  test: {
    include: ['tests/security/**/*.test.ts'],
    testTimeout: 30_000, // Security tests may need more time
    sequence: { concurrent: false }, // Run sequentially for accurate rate limit testing
  },
});

// Run: npx vitest --config vitest.config.security.ts
```
