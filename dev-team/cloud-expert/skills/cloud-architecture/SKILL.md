---
name: Cloud Architecture
description: This skill should be used when the user asks about "multi-region deployment", "disaster recovery", "data replication", "service mesh", "load balancing", "high availability", "fault tolerance", "auto-scaling", "circuit breaker", or "cloud resilience". It covers multi-region architecture, disaster recovery, service mesh, and resilience patterns.
---

# Cloud Architecture

## Multi-Region Architecture

```
Active-Active:
┌─────────────────┐     ┌─────────────────┐
│   US-East-1     │     │   EU-West-1     │
│ ┌─────────────┐ │     │ ┌─────────────┐ │
│ │ ALB → ECS   │ │     │ │ ALB → ECS   │ │
│ │ Aurora (RW) │←──────→│ │ Aurora (RW) │ │
│ │ ElastiCache │ │     │ │ ElastiCache │ │
│ └─────────────┘ │     │ └─────────────┘ │
└────────┬────────┘     └────────┬────────┘
         │   Route 53 Latency    │
         └───────────────────────┘

Active-Passive:
┌─────────────────┐     ┌─────────────────┐
│   US-East-1     │     │   US-West-2     │
│   (PRIMARY)     │     │   (STANDBY)     │
│ ┌─────────────┐ │     │ ┌─────────────┐ │
│ │ ALB → ECS   │ │     │ │ ALB → ECS   │ │
│ │ Aurora (RW) │──────→│ │ Aurora (RO) │ │
│ │ ElastiCache │ │     │ │ ElastiCache │ │
│ └─────────────┘ │     │ └─────────────┘ │
└─────────────────┘     └─────────────────┘
  Failover: Promote Aurora replica, scale up ECS, update Route 53
```

## Disaster Recovery Tiers

```
┌──────────────────────────────────────────────────────────┐
│ Strategy       │ RTO      │ RPO     │ Cost │ Complexity │
├──────────────────────────────────────────────────────────┤
│ Backup/Restore │ Hours    │ Hours   │ $    │ Low        │
│ Pilot Light    │ 10-30min │ Minutes │ $$   │ Medium     │
│ Warm Standby   │ Minutes  │ Seconds │ $$$  │ High       │
│ Active-Active  │ ~0       │ ~0      │ $$$$ │ Very High  │
└──────────────────────────────────────────────────────────┘
```

## Auto-Scaling Patterns

```yaml
# Kubernetes Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 120
```

```typescript
// AWS Auto Scaling with predictive scaling
const scaling = service.autoScaleTaskCount({
  minCapacity: 2,
  maxCapacity: 50,
});

// Target tracking — maintain 70% CPU
scaling.scaleOnCpuUtilization('CpuScaling', {
  targetUtilizationPercent: 70,
  scaleInCooldown: Duration.seconds(300),
  scaleOutCooldown: Duration.seconds(60),
});

// Scheduled scaling — pre-scale for known traffic patterns
scaling.scaleOnSchedule('MorningScaleUp', {
  schedule: appscaling.Schedule.cron({ hour: '8', minute: '0' }),
  minCapacity: 10,
});

scaling.scaleOnSchedule('NightScaleDown', {
  schedule: appscaling.Schedule.cron({ hour: '22', minute: '0' }),
  minCapacity: 2,
});
```

## Service Mesh (Istio)

```yaml
# Traffic splitting for canary deployment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-service
spec:
  hosts: [api-service]
  http:
    - route:
        - destination:
            host: api-service
            subset: stable
          weight: 90
        - destination:
            host: api-service
            subset: canary
          weight: 10
      timeout: 5s
      retries:
        attempts: 3
        retryOn: 5xx,reset,connect-failure

---
# mTLS between services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

## References

- [Resilience Patterns](references/resilience-patterns.md) — Circuit breakers, bulkheads, retry strategies, health checks, graceful degradation, chaos engineering.
- [Data Replication](references/data-replication.md) — Cross-region replication for Aurora, DynamoDB, S3, ElastiCache; consistency models, conflict resolution.
