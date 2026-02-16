# Logging Patterns

## Log Aggregation with Loki

```yaml
# Promtail configuration — ship logs to Loki
# promtail-config.yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    pipeline_stages:
      - json:
          expressions:
            level: level
            msg: msg
            traceId: traceId
      - labels:
          level:
      - timestamp:
          source: time
          format: RFC3339Nano
```

```
# LogQL queries in Grafana
# Find errors for a service
{app="my-service"} |= "error" | json | level="error"

# Count errors by type
sum by (error_code) (
  count_over_time({app="my-service"} | json | level="error" [5m])
)

# Trace correlation — find logs for a specific trace
{app=~".+"} | json | traceId="abc123def456"

# Slow request logs
{app="my-service"} | json | duration_ms > 1000
```

## Correlation IDs

```typescript
// Middleware to propagate trace context through all logs
import { randomUUID } from 'node:crypto';
import { AsyncLocalStorage } from 'node:async_hooks';

const asyncStorage = new AsyncLocalStorage<{ requestId: string; traceId: string }>();

// Express middleware
app.use((req, res, next) => {
  const requestId = req.headers['x-request-id'] as string || randomUUID();
  const traceId = req.headers['traceparent']?.split('-')[1] || randomUUID();

  res.setHeader('x-request-id', requestId);

  asyncStorage.run({ requestId, traceId }, () => next());
});

// Logger that auto-includes correlation IDs
function getLogger() {
  const store = asyncStorage.getStore();
  return logger.child({
    requestId: store?.requestId,
    traceId: store?.traceId,
  });
}

// Usage — every log automatically includes requestId and traceId
app.get('/api/orders', async (req, res) => {
  const log = getLogger();
  log.info('Fetching orders');
  const orders = await orderService.findAll();
  log.info({ count: orders.length }, 'Orders fetched');
  res.json(orders);
});
```

## Log Levels Strategy

```
┌────────────────────────────────────────────────────────────┐
│ Level │ When to Use                          │ Environment │
├────────────────────────────────────────────────────────────┤
│ FATAL │ Process cannot continue              │ All         │
│ ERROR │ Operation failed, needs attention     │ All         │
│ WARN  │ Unexpected but recoverable           │ All         │
│ INFO  │ Business events, state transitions   │ All         │
│ DEBUG │ Detailed flow, variable values       │ Dev/Staging │
│ TRACE │ Very detailed, per-iteration         │ Dev only    │
└────────────────────────────────────────────────────────────┘

Rules:
- ERROR: Something broke that needs human attention
- WARN:  Something unexpected that the system handled
- INFO:  Significant business events (order placed, user registered)
- DEBUG: Developer-useful context (query results, config loaded)
- Never log: Passwords, tokens, PII, credit card numbers
```

## ELK Stack Configuration

```yaml
# Filebeat → Elasticsearch → Kibana
# filebeat.yml
filebeat.inputs:
  - type: container
    paths: ['/var/log/containers/*.log']
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
      - decode_json_fields:
          fields: ['message']
          target: ''
          overwrite_keys: true

output.elasticsearch:
  hosts: ['elasticsearch:9200']
  indices:
    - index: "app-logs-%{+yyyy.MM.dd}"
      when.contains:
        kubernetes.labels.type: "application"
    - index: "system-logs-%{+yyyy.MM.dd}"

# Index lifecycle management
setup.ilm:
  enabled: true
  policy_name: "logs-policy"
  # Hot: 7 days → Warm: 30 days → Cold: 90 days → Delete
```

## Log Cost Management

```yaml
# Strategies to control log volume and cost:

# 1. Sample verbose logs
pipeline_stages:
  - match:
      selector: '{level="debug"}'
      stages:
        - sampling:
            rate: 10  # Keep 1 in 10 debug logs

# 2. Drop noisy logs
  - match:
      selector: '{app="load-balancer"}'
      stages:
        - drop:
            expression: "health_check"

# 3. Retention policies
# Hot storage: 7 days (fast SSD)
# Warm storage: 30 days (standard)
# Cold storage: 90 days (S3/GCS)
# Archive: 1 year (compliance only)

# 4. Structured logging reduces parsing cost
# JSON logs are 2-3x cheaper to process than unstructured text
# Pre-extract labels at ingestion time, not query time
```
