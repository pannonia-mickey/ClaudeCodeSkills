---
name: Performance Testing
description: This skill should be used when the user asks about "load testing", "stress testing", "k6 testing", "Artillery testing", "benchmarking", "performance budgets", "SLA testing", "throughput testing", "latency testing", or "performance profiling". It covers load testing with k6 and Artillery, benchmarking, performance budgets, and profiling.
---

# Performance Testing

## Load Testing with k6

```javascript
// k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const latency = new Trend('api_latency');

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // Ramp up
    { duration: '5m', target: 50 },   // Sustain
    { duration: '2m', target: 100 },  // Peak
    { duration: '1m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    errors: ['rate<0.01'],            // <1% error rate
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('http://localhost:3000/api/products');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'body has products': (r) => JSON.parse(r.body).data.length > 0,
  });
  errorRate.add(res.status !== 200);
  latency.add(res.timings.duration);
  sleep(1);
}
```

## Stress Testing Pattern

```javascript
// k6/stress-test.js — find breaking point
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 },  // Push beyond expected load
    { duration: '5m', target: 300 },
    { duration: '5m', target: 0 },    // Recovery
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'], // Relaxed for stress
  },
};
```

## Artillery Load Testing

```yaml
# artillery/load-test.yml
config:
  target: http://localhost:3000
  phases:
    - duration: 120
      arrivalRate: 10
      name: Warm up
    - duration: 300
      arrivalRate: 50
      name: Sustained load
    - duration: 120
      arrivalRate: 100
      name: Peak
  defaults:
    headers:
      Authorization: "Bearer {{ $processEnvironment.TEST_TOKEN }}"
  plugins:
    expect: {}
    metrics-by-endpoint: {}

scenarios:
  - name: Browse and purchase flow
    weight: 70
    flow:
      - get:
          url: /api/products
          expect:
            - statusCode: 200
            - hasProperty: data
      - think: 2
      - post:
          url: /api/cart/items
          json:
            productId: "{{ $randomString() }}"
            quantity: 1
          expect:
            - statusCode: 201
      - think: 3
      - post:
          url: /api/checkout
          expect:
            - statusCode: 200

  - name: API health check
    weight: 30
    flow:
      - get:
          url: /api/health
          expect:
            - statusCode: 200
```

## Benchmarking with Vitest

```typescript
import { bench, describe } from 'vitest';

describe('String processing benchmarks', () => {
  bench('regex replace', () => {
    'hello-world-foo-bar'.replace(/-./g, (m) => m[1].toUpperCase());
  });

  bench('split and join', () => {
    'hello-world-foo-bar'
      .split('-')
      .map((w, i) => (i ? w[0].toUpperCase() + w.slice(1) : w))
      .join('');
  });
});

// Run: npx vitest bench
```

## Performance Budgets

```typescript
// Lighthouse CI configuration
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/products'],
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
        'interactive': ['error', { maxNumericValue: 3500 }],
      },
    },
  },
};
```

## References

- [Load Testing Patterns](references/load-testing.md) — k6 scenarios, Artillery custom functions, distributed testing, CI integration.
- [Profiling Patterns](references/profiling-patterns.md) — Node.js profiling, memory leak detection, CPU profiling, APM integration.
