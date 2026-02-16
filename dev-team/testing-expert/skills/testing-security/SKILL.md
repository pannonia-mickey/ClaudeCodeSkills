---
name: Security Testing
description: This skill should be used when the user asks about "security testing", "OWASP testing", "SAST", "DAST", "dependency scanning", "penetration testing", "security regression", "vulnerability scanning", "ZAP testing", or "security audit automation". It covers SAST/DAST tools, dependency scanning, OWASP ZAP integration, and security regression testing.
---

# Security Testing

## SAST — Static Application Security Testing

```yaml
# GitHub Actions — ESLint security rules + Semgrep
security-scan:
  steps:
    - name: ESLint security plugin
      run: npx eslint --config eslint-security.config.js src/

    - name: Semgrep SAST scan
      uses: semgrep/semgrep-action@v1
      with:
        config: >-
          p/javascript
          p/typescript
          p/nodejs
          p/owasp-top-ten
```

```javascript
// eslint-security.config.js
import security from 'eslint-plugin-security';

export default [
  {
    plugins: { security },
    rules: {
      'security/detect-object-injection': 'warn',
      'security/detect-non-literal-regexp': 'warn',
      'security/detect-unsafe-regex': 'error',
      'security/detect-buffer-noassert': 'error',
      'security/detect-eval-with-expression': 'error',
      'security/detect-no-csrf-before-method-override': 'error',
      'security/detect-possible-timing-attacks': 'warn',
      'security/detect-child-process': 'warn',
    },
  },
];
```

## Dependency Scanning

```yaml
# npm audit in CI
- name: Security audit
  run: |
    npm audit --audit-level=high
    # Fail on high/critical vulnerabilities only

# Snyk integration
- name: Snyk security scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high

# Socket.dev — supply chain attack detection
- name: Socket security
  uses: SocketDev/socket-security-action@v1
```

```json
// .nsprc — audit exceptions for known false positives
{
  "exceptions": [
    "https://github.com/advisories/GHSA-xxxx-yyyy-zzzz"
  ]
}
```

## DAST — Dynamic Application Security Testing

```typescript
// OWASP ZAP integration test
import { ZapClient } from 'zaproxy';

describe('Security scan', () => {
  const zap = new ZapClient({ apiKey: process.env.ZAP_API_KEY, proxy: 'http://localhost:8080' });

  it('should pass ZAP baseline scan', async () => {
    // Spider the application
    await zap.spider.scan({ url: 'http://localhost:3000' });
    await waitForSpider(zap);

    // Active scan
    await zap.ascan.scan({ url: 'http://localhost:3000' });
    await waitForScan(zap);

    // Check alerts
    const alerts = await zap.core.alerts({ baseurl: 'http://localhost:3000' });
    const highAlerts = alerts.filter((a) => a.risk >= 3); // High/Critical
    expect(highAlerts).toEqual([]);
  });
});
```

```yaml
# ZAP baseline scan in CI
- name: ZAP Baseline Scan
  uses: zaproxy/action-baseline@v0.13.0
  with:
    target: http://localhost:3000
    rules_file_name: zap-rules.tsv
    fail_action: true
```

## Security Regression Tests

```typescript
// Security-focused test suite
describe('Security regression tests', () => {
  // Authentication
  it('should reject requests without auth token', async () => {
    const res = await request(app).get('/api/users');
    expect(res.status).toBe(401);
  });

  it('should reject expired tokens', async () => {
    const expiredToken = generateToken({ exp: Math.floor(Date.now() / 1000) - 3600 });
    const res = await request(app)
      .get('/api/users')
      .set('Authorization', `Bearer ${expiredToken}`);
    expect(res.status).toBe(401);
  });

  // Authorization
  it('should prevent horizontal privilege escalation', async () => {
    const userToken = await loginAs('user@test.com');
    const res = await request(app)
      .get('/api/users/other-user-id/private-data')
      .set('Authorization', `Bearer ${userToken}`);
    expect(res.status).toBe(403);
  });

  // Input validation
  it('should reject SQL injection attempts', async () => {
    const res = await request(app)
      .get('/api/users?search=\' OR 1=1 --');
    expect(res.status).toBe(400);
  });

  // XSS prevention
  it('should sanitize user input in responses', async () => {
    await createUser({ name: '<script>alert("xss")</script>' });
    const res = await request(app).get('/api/users');
    expect(res.text).not.toContain('<script>');
  });

  // Security headers
  it('should set security headers', async () => {
    const res = await request(app).get('/');
    expect(res.headers['x-content-type-options']).toBe('nosniff');
    expect(res.headers['x-frame-options']).toBe('DENY');
    expect(res.headers['strict-transport-security']).toBeDefined();
    expect(res.headers['content-security-policy']).toBeDefined();
  });

  // Rate limiting
  it('should enforce rate limits', async () => {
    const requests = Array.from({ length: 101 }, () =>
      request(app).get('/api/auth/login')
    );
    const responses = await Promise.all(requests);
    const rateLimited = responses.filter((r) => r.status === 429);
    expect(rateLimited.length).toBeGreaterThan(0);
  });
});
```

## References

- [Security Scanning](references/security-scanning.md) — Semgrep rules, Snyk policies, container scanning, secrets detection, CI pipeline.
- [Penetration Testing](references/penetration-testing.md) — OWASP Top 10 test cases, API security testing, authentication bypass tests, injection testing.
