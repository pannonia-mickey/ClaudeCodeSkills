---
name: Monitoring & Observability
description: This skill should be used when the user asks about "Prometheus", "Grafana", "OpenTelemetry", "observability", "distributed tracing", "structured logging", "alerting", "SLI", "SLO", "SLA", "dashboards", "Loki", "Tempo", "Jaeger", or "metrics collection". It covers the three pillars of observability, SLO-based alerting, and dashboard design.
---

# Monitoring & Observability

## Three Pillars of Observability

```
┌────────────────────────────────────────────────────────┐
│                  OpenTelemetry Collector                │
│           (Unified telemetry pipeline)                 │
├──────────────┬──────────────┬──────────────────────────┤
│   Metrics    │    Logs      │     Traces               │
│  Prometheus  │  Loki / ELK  │  Tempo / Jaeger          │
│  → Grafana   │  → Grafana   │  → Grafana               │
└──────────────┴──────────────┴──────────────────────────┘
```

## Prometheus Metrics

```yaml
# Kubernetes ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-service
spec:
  selector:
    matchLabels:
      app: my-service
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

```typescript
// Express metrics middleware with prom-client
import { Registry, Counter, Histogram, collectDefaultMetrics } from 'prom-client';

const registry = new Registry();
collectDefaultMetrics({ register: registry });

const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [registry],
});

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [registry],
});

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    const route = req.route?.path || req.path;
    httpRequestsTotal.inc({ method: req.method, route, status: res.statusCode });
    end({ method: req.method, route, status: res.statusCode });
  });
  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', registry.contentType);
  res.send(await registry.metrics());
});
```

## SLI / SLO / SLA

```yaml
# SLO definitions
slos:
  - name: API Availability
    sli: sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
    target: 99.9%       # 43.8 min downtime/month
    error_budget: 0.1%

  - name: API Latency
    sli: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
    target: "p99 < 500ms"

  - name: Data Processing
    sli: sum(rate(jobs_completed_total[1h])) / sum(rate(jobs_submitted_total[1h]))
    target: 99.5%
```

## Structured Logging

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: {
    service: 'my-service',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION,
  },
});

// Structured log with trace context
logger.info({
  msg: 'Order processed',
  orderId: order.id,
  userId: order.userId,
  total: order.total,
  duration_ms: processingTime,
  traceId: span.spanContext().traceId,
});
```

## Alerting Rules

```yaml
# PrometheusRule — SLO-based alerting
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-alerts
spec:
  groups:
    - name: slo.rules
      rules:
        # Error budget burn rate alert
        - alert: HighErrorBudgetBurn
          expr: |
            (
              sum(rate(http_requests_total{status=~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            ) > (14.4 * 0.001)
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Error budget burning 14.4x faster than allowed"
            dashboard: "https://grafana.internal/d/slo-overview"

        - alert: HighLatency
          expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
          for: 10m
          labels:
            severity: warning
```

## References

- [Dashboard Design](references/dashboard-design.md) — RED/USE methods, Grafana dashboard templates, service overview dashboards, business metrics.
- [Logging Patterns](references/logging-patterns.md) — Log aggregation with Loki, correlation IDs, log levels strategy, ELK stack, cost management.
