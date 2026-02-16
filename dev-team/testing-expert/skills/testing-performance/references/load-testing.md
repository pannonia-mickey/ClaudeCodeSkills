# Load Testing Patterns

## k6 Advanced Scenarios

```javascript
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// Load test data from file
const users = new SharedArray('users', function () {
  return JSON.parse(open('./test-data/users.json'));
});

export const options = {
  scenarios: {
    // Simulate different user behaviors simultaneously
    browse: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '1m', target: 0 },
      ],
      exec: 'browseProducts',
    },
    purchase: {
      executor: 'constant-arrival-rate',
      rate: 10,            // 10 purchases per second
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50,
      exec: 'completePurchase',
    },
    api_spike: {
      executor: 'ramping-arrival-rate',
      startRate: 1,
      stages: [
        { duration: '1m', target: 50 },   // Sudden spike
        { duration: '2m', target: 50 },
        { duration: '1m', target: 1 },    // Back to normal
      ],
      preAllocatedVUs: 100,
      exec: 'apiCalls',
    },
  },
  thresholds: {
    'http_req_duration{scenario:browse}': ['p(95)<800'],
    'http_req_duration{scenario:purchase}': ['p(95)<2000'],
    'http_req_failed': ['rate<0.01'],
  },
};

export function browseProducts() {
  group('Browse catalog', () => {
    const res = http.get('http://localhost:3000/api/products?page=1');
    check(res, { 'products loaded': (r) => r.status === 200 });
    sleep(Math.random() * 3 + 1); // 1-4 seconds think time

    const products = JSON.parse(res.body).data;
    if (products.length > 0) {
      const detail = http.get(`http://localhost:3000/api/products/${products[0].id}`);
      check(detail, { 'detail loaded': (r) => r.status === 200 });
    }
  });
  sleep(2);
}

export function completePurchase() {
  const user = users[Math.floor(Math.random() * users.length)];

  group('Purchase flow', () => {
    // Login
    const login = http.post('http://localhost:3000/api/auth/login', JSON.stringify({
      email: user.email,
      password: user.password,
    }), { headers: { 'Content-Type': 'application/json' } });

    check(login, { 'logged in': (r) => r.status === 200 });
    const token = JSON.parse(login.body).token;
    const headers = { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' };

    // Add to cart
    http.post('http://localhost:3000/api/cart/items',
      JSON.stringify({ productId: 'prod-1', quantity: 1 }),
      { headers }
    );

    // Checkout
    const checkout = http.post('http://localhost:3000/api/checkout', null, { headers });
    check(checkout, { 'checkout success': (r) => r.status === 200 });
  });
}

export function apiCalls() {
  http.get('http://localhost:3000/api/health');
  sleep(0.1);
}
```

## Artillery Custom Functions

```javascript
// artillery/processor.js
module.exports = {
  generateUser(context, events, done) {
    context.vars.email = `loadtest+${Date.now()}@test.com`;
    context.vars.password = 'LoadTest123!';
    return done();
  },

  logResponse(req, res, context, events, done) {
    if (res.statusCode >= 400) {
      console.log(`Error ${res.statusCode}: ${res.body}`);
    }
    return done();
  },

  validateResponse(req, res, context, events, done) {
    const body = JSON.parse(res.body);
    if (!body.data || body.data.length === 0) {
      events.emit('counter', 'empty_responses', 1);
    }
    return done();
  },
};
```

```yaml
# artillery/config.yml — using custom functions
config:
  target: http://localhost:3000
  processor: ./processor.js
  phases:
    - duration: 300
      arrivalRate: 20

scenarios:
  - flow:
      - function: generateUser
      - post:
          url: /api/auth/register
          json:
            email: "{{ email }}"
            password: "{{ password }}"
          afterResponse: logResponse
```

## Distributed Load Testing

```yaml
# k6 Cloud / Grafana Cloud k6
# k6 run --out cloud script.js

# Self-hosted distributed with k6-operator (Kubernetes)
# k6-operator.yml
apiVersion: k6.io/v1alpha1
kind: K6
metadata:
  name: load-test
spec:
  parallelism: 4           # 4 pods running the test
  script:
    configMap:
      name: k6-test-script
  runner:
    resources:
      limits:
        cpu: "500m"
        memory: "256Mi"
```

## CI Integration

```yaml
# GitHub Actions — performance regression detection
performance-test:
  runs-on: ubuntu-latest
  services:
    app:
      image: myapp:${{ github.sha }}
      ports: ['3000:3000']
  steps:
    - uses: actions/checkout@v4

    - name: Install k6
      run: |
        sudo gpg -k
        sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
          --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D68
        echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
          | sudo tee /etc/apt/sources.list.d/k6.list
        sudo apt-get update && sudo apt-get install k6

    - name: Run load test
      run: k6 run --out json=results.json k6/load-test.js

    - name: Check thresholds
      run: |
        PASSED=$(jq '.root_group.checks // 0 | length' results.json)
        echo "Checks passed: $PASSED"

    - uses: actions/upload-artifact@v4
      if: always()
      with:
        name: k6-results
        path: results.json
```

## Soak Testing

```javascript
// Long-running test to detect memory leaks and degradation
export const options = {
  stages: [
    { duration: '5m', target: 50 },     // Ramp up
    { duration: '4h', target: 50 },     // Sustained load
    { duration: '5m', target: 0 },      // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<1000'],
    // Track degradation over time
    'http_req_duration{window:0-30m}': ['p(95)<500'],
    'http_req_duration{window:3h30m-4h}': ['p(95)<700'], // Allow 40% degradation max
  },
};
```
