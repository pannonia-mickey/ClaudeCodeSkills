# Dashboard Design

## RED Method (Request-oriented)

```
Rate:     Requests per second
Errors:   Failed requests per second
Duration: Request latency distribution

# Prometheus queries for RED dashboard
# Rate
sum(rate(http_requests_total[5m])) by (service)

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)

# Duration (p50, p95, p99)
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
```

## USE Method (Resource-oriented)

```
Utilization: % of resource capacity used
Saturation:  Amount of queued/waiting work
Errors:      Count of error events

# CPU
Utilization: rate(process_cpu_seconds_total[5m])
Saturation:  process_runqueue_length (if available)

# Memory
Utilization: process_resident_memory_bytes / machine_memory_bytes
Saturation:  rate(node_vmstat_pgmajfault[5m])

# Disk
Utilization: node_filesystem_avail_bytes / node_filesystem_size_bytes
Saturation:  rate(node_disk_io_time_weighted_seconds_total[5m])

# Network
Utilization: rate(node_network_receive_bytes_total[5m])
Errors:      rate(node_network_receive_errs_total[5m])
```

## Grafana Dashboard Template (JSON)

```json
{
  "dashboard": {
    "title": "Service Overview",
    "templating": {
      "list": [
        {
          "name": "service",
          "type": "query",
          "query": "label_values(http_requests_total, service)",
          "refresh": 2
        }
      ]
    },
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "targets": [{
          "expr": "sum(rate(http_requests_total{service=\"$service\"}[5m]))",
          "legendFormat": "requests/s"
        }],
        "gridPos": { "h": 8, "w": 8, "x": 0, "y": 0 }
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [{
          "expr": "sum(rate(http_requests_total{service=\"$service\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{service=\"$service\"}[5m])) * 100"
        }],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                { "color": "green", "value": 0 },
                { "color": "yellow", "value": 1 },
                { "color": "red", "value": 5 }
              ]
            },
            "unit": "percent"
          }
        },
        "gridPos": { "h": 8, "w": 8, "x": 8, "y": 0 }
      },
      {
        "title": "Latency Distribution",
        "type": "timeseries",
        "targets": [
          { "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le))", "legendFormat": "p50" },
          { "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le))", "legendFormat": "p95" },
          { "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le))", "legendFormat": "p99" }
        ],
        "gridPos": { "h": 8, "w": 8, "x": 16, "y": 0 }
      }
    ]
  }
}
```

## Service Map Dashboard

```
# Grafana with Tempo — auto-discovered service topology
# Shows: service → service call graph with latency/error decorations

# Key panels for service map dashboard:
1. Service Topology Graph (Tempo/Jaeger data source)
2. Inter-service latency heatmap
3. Dependency error rates
4. Top N slowest traces table
```

## Business Metrics Dashboard

```
# Track business KPIs alongside technical metrics
# Panel examples:

# Revenue per minute
sum(rate(order_total_amount[5m])) * 60

# Conversion funnel
  Page Views:     sum(increase(page_views_total{page="product"}[1h]))
  Add to Cart:    sum(increase(cart_additions_total[1h]))
  Checkout Start: sum(increase(checkout_started_total[1h]))
  Purchase:       sum(increase(orders_completed_total[1h]))

# Active users (custom metric)
count(count by (user_id) (http_requests_total{path!="/health"}))

# Feature adoption
sum(rate(feature_usage_total{feature="new_checkout"}[1h]))
/
sum(rate(feature_usage_total[1h]))
```

## Alert Dashboard

```yaml
# Alert overview panel queries
# Firing alerts
ALERTS{alertstate="firing"}

# Alert history (with Alertmanager)
# Group by severity
count(ALERTS{alertstate="firing"}) by (severity)

# Mean time to acknowledge
# Track in PagerDuty/Opsgenie → export to Prometheus
```
