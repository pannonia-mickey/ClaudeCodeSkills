# Profiling Patterns

## Node.js CPU Profiling

```typescript
// Built-in profiler
// node --prof app.js
// node --prof-process isolate-*.log > profile.txt

// Programmatic profiling with v8-profiler-next
import { Session } from 'node:inspector/promises';

async function profileFunction(fn: () => Promise<void>) {
  const session = new Session();
  session.connect();

  await session.post('Profiler.enable');
  await session.post('Profiler.start');

  await fn();

  const { profile } = await session.post('Profiler.stop');
  writeFileSync('cpu-profile.cpuprofile', JSON.stringify(profile));
  // Open in Chrome DevTools → Performance tab

  session.disconnect();
}

// Clinic.js — comprehensive profiling
// npx clinic doctor -- node app.js
// npx clinic flame -- node app.js     # flame graphs
// npx clinic bubbleprof -- node app.js # async analysis
```

## Memory Leak Detection

```typescript
// Heap snapshot comparison
import { writeHeapSnapshot } from 'node:v8';

// Take snapshots at intervals
let snapshotCount = 0;
setInterval(() => {
  if (snapshotCount < 3) {
    const filename = writeHeapSnapshot();
    console.log(`Heap snapshot written to ${filename}`);
    snapshotCount++;
  }
}, 60_000); // Every minute

// Compare snapshots in Chrome DevTools:
// 1. Load snapshot 1 and 2
// 2. Select "Comparison" view
// 3. Sort by "# Delta" to find growing allocations

// Programmatic memory monitoring
function monitorMemory(intervalMs = 10_000) {
  setInterval(() => {
    const usage = process.memoryUsage();
    console.log({
      rss: `${(usage.rss / 1024 / 1024).toFixed(1)} MB`,
      heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(1)} MB`,
      heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(1)} MB`,
      external: `${(usage.external / 1024 / 1024).toFixed(1)} MB`,
    });
  }, intervalMs);
}
```

## Memory Leak Test Pattern

```typescript
// Test for memory leaks in long-running operations
import { describe, it, expect } from 'vitest';

describe('Memory leak tests', () => {
  it('should not leak memory in event processing', async () => {
    // Force GC before measuring
    if (global.gc) global.gc();
    const before = process.memoryUsage().heapUsed;

    // Run operation many times
    for (let i = 0; i < 10_000; i++) {
      await processEvent({ id: i, data: 'x'.repeat(1000) });
    }

    // Force GC and measure after
    if (global.gc) global.gc();
    const after = process.memoryUsage().heapUsed;

    const growth = (after - before) / 1024 / 1024; // MB
    console.log(`Memory growth: ${growth.toFixed(2)} MB`);
    expect(growth).toBeLessThan(50); // Allow max 50MB growth
  });
});

// Run with: node --expose-gc node_modules/.bin/vitest
```

## Event Loop Monitoring

```typescript
// Detect event loop lag
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  console.log({
    min: `${(histogram.min / 1e6).toFixed(2)}ms`,
    max: `${(histogram.max / 1e6).toFixed(2)}ms`,
    mean: `${(histogram.mean / 1e6).toFixed(2)}ms`,
    p99: `${(histogram.percentile(99) / 1e6).toFixed(2)}ms`,
  });
  histogram.reset();
}, 5000);

// Alert on high event loop lag
function checkEventLoopHealth() {
  const p99 = histogram.percentile(99) / 1e6;
  if (p99 > 100) {
    console.warn(`Event loop lag p99: ${p99.toFixed(1)}ms — investigate blocking operations`);
  }
}
```

## HTTP Response Time Tracking

```typescript
// Express middleware for response time monitoring
import { performance, PerformanceObserver } from 'node:perf_hooks';

function responseTimeMiddleware(req, res, next) {
  const start = performance.now();

  res.on('finish', () => {
    const duration = performance.now() - start;
    const route = req.route?.path || req.path;
    const method = req.method;

    // Log slow requests
    if (duration > 1000) {
      console.warn(`Slow request: ${method} ${route} — ${duration.toFixed(0)}ms`);
    }

    // Emit metric for monitoring
    metrics.histogram('http_request_duration_ms', duration, {
      method,
      route,
      status: res.statusCode,
    });
  });

  next();
}
```

## Database Query Profiling

```typescript
// Prisma query logging
const prisma = new PrismaClient({
  log: [
    { emit: 'event', level: 'query' },
  ],
});

prisma.$on('query', (e) => {
  if (e.duration > 100) {
    console.warn(`Slow query (${e.duration}ms): ${e.query}`);
    console.warn(`Params: ${e.params}`);
  }
});

// Test for N+1 queries
describe('UserService', () => {
  it('should fetch users without N+1', async () => {
    const queries: string[] = [];
    prisma.$on('query', (e) => queries.push(e.query));

    await userService.getUsersWithOrders();

    const selectQueries = queries.filter((q) => q.startsWith('SELECT'));
    expect(selectQueries.length).toBeLessThanOrEqual(2); // 1 for users, 1 for orders
  });
});
```

## APM Integration

```typescript
// OpenTelemetry setup for performance monitoring
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://localhost:4318/v1/traces',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingPaths: ['/health'],
      },
      '@opentelemetry/instrumentation-express': {},
      '@opentelemetry/instrumentation-pg': {
        enhancedDatabaseReporting: true,
      },
    }),
  ],
});

sdk.start();
```
