# Incident Management

## Runbook Template

```markdown
# Runbook: Service Name — Issue Type

## Overview
- **Service**: service-name
- **Owner**: team-name
- **Escalation**: #incident-channel → on-call → engineering-lead

## Detection
- **Alert**: `ServiceHighErrorRate` fires when error rate > 5% for 5m
- **Dashboard**: https://grafana.internal/d/service-overview

## Diagnosis
1. Check service health: `kubectl get pods -n production -l app=service-name`
2. Check recent deployments: `kubectl rollout history deployment/service-name`
3. Check error logs: `kubectl logs -n production -l app=service-name --since=10m | grep ERROR`
4. Check dependencies:
   - Database: `pg_isready -h db-host`
   - Redis: `redis-cli -h redis-host ping`
   - Upstream API: `curl -s -o /dev/null -w "%{http_code}" https://upstream/health`

## Mitigation
### Rollback deployment
```bash
kubectl rollout undo deployment/service-name -n production
kubectl rollout status deployment/service-name -n production
```

### Scale up (if load-related)
```bash
kubectl scale deployment/service-name --replicas=6 -n production
```

### Restart pods (if memory leak suspected)
```bash
kubectl rollout restart deployment/service-name -n production
```

## Recovery Verification
1. Error rate returns to baseline (< 0.1%)
2. Latency p99 returns to baseline (< 500ms)
3. No new error log entries
4. Confirm in dashboard for 15 minutes
```

## Postmortem Template

```markdown
# Postmortem: [Title]

**Date**: YYYY-MM-DD
**Duration**: HH:MM start → HH:MM resolved (X hours)
**Severity**: SEV-X
**Impact**: X% of users affected, Y requests failed
**Authors**: [Names]

## Summary
One paragraph describing what happened, the impact, and how it was resolved.

## Timeline (UTC)
| Time  | Event                                      |
|-------|--------------------------------------------|
| 14:00 | Deployment v2.3.1 rolled out               |
| 14:05 | Error rate alert fires                     |
| 14:08 | On-call acknowledges, begins investigation |
| 14:15 | Root cause identified: DB connection leak  |
| 14:18 | Rollback to v2.3.0 initiated               |
| 14:22 | Service recovered, error rate normal       |

## Root Cause
Detailed technical explanation of what went wrong.

## What Went Well
- Alert fired within 5 minutes of issue
- Runbook was up to date
- Rollback procedure worked smoothly

## What Went Wrong
- DB connection leak not caught in code review
- No load testing on the changed code path
- Staging environment didn't reproduce (different connection pool size)

## Action Items
| Priority | Action                                    | Owner    | Due      |
|----------|-------------------------------------------|----------|----------|
| P1       | Add connection pool monitoring metric     | @dev     | 2025-02-01 |
| P1       | Add load test for auth endpoint           | @qa      | 2025-02-01 |
| P2       | Align staging DB pool config with prod    | @devops  | 2025-02-15 |
| P3       | Add connection leak detector to CI        | @dev     | 2025-03-01 |

## Lessons Learned
Key takeaways for the team.
```

## On-Call Rotation

```yaml
# PagerDuty-style rotation config
schedule:
  name: Backend On-Call
  type: weekly_rotation
  participants:
    - name: Alice
      escalation_delay: 15m
    - name: Bob
      escalation_delay: 15m
    - name: Carol
      escalation_delay: 15m
  overrides: [] # Vacation swaps
  escalation_policy:
    - level: 1
      targets: [current_on_call]
      timeout: 15m
    - level: 2
      targets: [engineering_lead]
      timeout: 30m
    - level: 3
      targets: [vp_engineering]

# On-call best practices:
# - Maximum 1 week rotation, minimum 2 people
# - Follow-the-sun for global teams
# - Compensate on-call time
# - Budget 30% on-call time for toil reduction
# - Review alert volume weekly — alert fatigue kills effectiveness
```

## Chaos Engineering

```yaml
# Litmus Chaos experiment — pod delete
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-delete-test
spec:
  appinfo:
    appns: production
    applabel: app=payment-service
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '30'
            - name: CHAOS_INTERVAL
              value: '10'
            - name: FORCE
              value: 'false'

# Steady-state hypothesis:
# - p99 latency stays below 500ms
# - Error rate stays below 1%
# - No data loss or corruption
```

## Escalation Matrix

```
┌──────────────────────────────────────────────────────┐
│ Category     │ L1 (On-Call)    │ L2 (Lead)    │ L3   │
├──────────────────────────────────────────────────────┤
│ App crash    │ Restart/rollback│ Debug fix    │ Arch │
│ Data issue   │ Read replica    │ DBA          │ CTO  │
│ Security     │ Isolate         │ SecOps       │ CISO │
│ Infra/Cloud  │ Failover        │ Platform     │ VP   │
│ Third-party  │ Monitor         │ Contact vendor│ Exec │
└──────────────────────────────────────────────────────┘
```
