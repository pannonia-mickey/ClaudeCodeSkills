# Resilience Patterns

## Circuit Breaker (with opossum)

```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(callExternalService, {
  timeout: 3000,           // 3s timeout per request
  errorThresholdPercentage: 50,  // Open at 50% errors
  resetTimeout: 30_000,    // Try half-open after 30s
  volumeThreshold: 10,     // Minimum 10 requests before tripping
  rollingCountTimeout: 10_000,   // 10s rolling window
});

breaker.on('open', () => logger.warn('Circuit opened — using fallback'));
breaker.on('halfOpen', () => logger.info('Circuit half-open — testing'));
breaker.on('close', () => logger.info('Circuit closed — recovered'));

breaker.fallback(() => ({
  source: 'cache',
  data: getCachedData(),
}));

// Usage
const result = await breaker.fire(requestParams);

// Prometheus metrics
breaker.on('success', () => metrics.increment('circuit_breaker_success'));
breaker.on('failure', () => metrics.increment('circuit_breaker_failure'));
breaker.on('reject', () => metrics.increment('circuit_breaker_rejected'));
```

## Bulkhead Pattern

```typescript
// Isolate critical from non-critical operations
import PQueue from 'p-queue';

// Separate thread pools (queues) for different operations
const criticalQueue = new PQueue({ concurrency: 50 });
const nonCriticalQueue = new PQueue({ concurrency: 10 });
const backgroundQueue = new PQueue({ concurrency: 5 });

// Critical operations get dedicated capacity
app.post('/api/payments', async (req, res) => {
  const result = await criticalQueue.add(() => processPayment(req.body));
  res.json(result);
});

// Non-critical operations can't starve critical ones
app.get('/api/recommendations', async (req, res) => {
  const result = await nonCriticalQueue.add(() => getRecommendations(req.query));
  res.json(result);
});
```

## Retry with Exponential Backoff

```typescript
interface RetryConfig {
  maxRetries: number;
  baseDelay: number;
  maxDelay: number;
  retryableErrors: string[];
}

async function withRetry<T>(
  fn: () => Promise<T>,
  config: RetryConfig = { maxRetries: 3, baseDelay: 1000, maxDelay: 30000, retryableErrors: [] }
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      // Don't retry non-retryable errors
      if (config.retryableErrors.length > 0 &&
          !config.retryableErrors.includes(lastError.name)) {
        throw error;
      }

      if (attempt === config.maxRetries) break;

      // Exponential backoff with full jitter
      const delay = Math.min(
        config.maxDelay,
        config.baseDelay * Math.pow(2, attempt)
      );
      const jitter = Math.random() * delay;
      await new Promise(r => setTimeout(r, jitter));

      logger.warn({
        attempt: attempt + 1,
        maxRetries: config.maxRetries,
        delay: Math.round(jitter),
        error: lastError.message,
      }, 'Retrying operation');
    }
  }

  throw lastError!;
}
```

## Health Check Patterns

```typescript
// Multi-level health checks
app.get('/health', async (req, res) => {
  // Basic liveness — is the process running?
  res.json({ status: 'ok' });
});

app.get('/health/ready', async (req, res) => {
  // Readiness — can this instance handle traffic?
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkExternalApi(),
  ]);

  const results = {
    database: checks[0].status === 'fulfilled',
    redis: checks[1].status === 'fulfilled',
    externalApi: checks[2].status === 'fulfilled',
  };

  const healthy = Object.values(results).every(Boolean);
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'ready' : 'degraded',
    checks: results,
  });
});

// Kubernetes probes
// livenessProbe:  → /health (restart if unhealthy)
// readinessProbe: → /health/ready (remove from LB if not ready)
// startupProbe:   → /health (give time to start, then switch to liveness)
```

## Graceful Degradation

```typescript
// Feature-level degradation based on system health
class FeatureGates {
  private degradationLevel: 'normal' | 'degraded' | 'critical' = 'normal';

  setLevel(level: typeof this.degradationLevel) {
    this.degradationLevel = level;
    logger.warn({ level }, 'Degradation level changed');
  }

  isEnabled(feature: string): boolean {
    const featureMap: Record<string, string[]> = {
      'recommendations': ['normal'],           // First to disable
      'search-suggestions': ['normal'],
      'analytics-tracking': ['normal', 'degraded'],
      'user-reviews': ['normal', 'degraded'],
      'core-checkout': ['normal', 'degraded', 'critical'], // Last to disable
      'order-processing': ['normal', 'degraded', 'critical'],
    };

    return featureMap[feature]?.includes(this.degradationLevel) ?? false;
  }
}

// Auto-degrade based on error rates
if (errorRate > 0.1) featureGates.setLevel('degraded');
if (errorRate > 0.3) featureGates.setLevel('critical');
```

## Chaos Engineering Principles

```
Steady State Hypothesis:
  Define what "healthy" looks like with measurable metrics:
  - p99 latency < 500ms
  - Error rate < 0.1%
  - Throughput > 1000 req/s

Experiments (start small, increase blast radius):
  Level 1: Kill a single pod/instance
  Level 2: Inject latency (100ms → 500ms → 2s)
  Level 3: Fail a dependency (database, cache, API)
  Level 4: Network partition between services
  Level 5: AZ failure simulation
  Level 6: Region failover test

Tools:
  - Litmus Chaos (Kubernetes)
  - AWS Fault Injection Simulator
  - Gremlin (commercial)
  - Chaos Monkey (Netflix)
  - toxiproxy (network-level chaos)
```
