---
name: Cloud Mastery
description: This skill should be used when the user asks about "well-architected framework", "cloud-native principles", "microservices architecture", "event-driven architecture", "cloud cost optimization", "cloud migration", or "cloud design patterns". It covers cloud architecture fundamentals, design patterns, and cost optimization strategies.
---

# Cloud Mastery

## Well-Architected Framework Pillars

```
┌────────────────────────────────────────────────────────────┐
│ Pillar                │ Key Practices                      │
├────────────────────────────────────────────────────────────┤
│ Operational Excellence│ IaC, CI/CD, runbooks, observability│
│ Security              │ IAM, encryption, network isolation │
│ Reliability           │ Multi-AZ, auto-scaling, DR         │
│ Performance           │ Right-sizing, caching, CDN         │
│ Cost Optimization     │ Reserved, spot, right-sizing       │
│ Sustainability        │ Efficient resources, managed svcs  │
└────────────────────────────────────────────────────────────┘
```

## Cloud-Native Design Patterns

```typescript
// Circuit Breaker — prevent cascade failures
class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure > 30_000) {
        this.state = 'half-open'; // Try one request
      } else {
        throw new Error('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();
      this.failures = 0;
      this.state = 'closed';
      return result;
    } catch (error) {
      this.failures++;
      this.lastFailure = Date.now();
      if (this.failures >= 5) this.state = 'open';
      throw error;
    }
  }
}

// Bulkhead — isolate failures to specific components
// Run critical operations in separate thread pools / Lambda functions
// If non-critical service is overwhelmed, critical services continue

// Retry with exponential backoff + jitter
async function retryWithBackoff<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      const delay = Math.min(1000 * 2 ** i, 30_000);
      const jitter = delay * 0.5 * Math.random();
      await new Promise((r) => setTimeout(r, delay + jitter));
    }
  }
  throw new Error('Unreachable');
}
```

## Event-Driven Architecture

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│ Producer │────→│ Event Bus    │────→│ Consumer A   │
│ (Order   │     │ (EventBridge │     │ (Inventory)  │
│  Service)│     │  / SNS)      │────→│ Consumer B   │
└──────────┘     └──────────────┘     │ (Billing)    │
                                      │ Consumer C   │
                                      │ (Notify)     │
                                      └──────────────┘
Benefits:
- Loose coupling between services
- Independent scaling per consumer
- Easy to add new consumers
- Natural audit trail
```

## Cost Optimization Strategy

```
Priority 1 — Quick Wins (days):
  □ Delete unused resources (EBS volumes, old snapshots, idle LBs)
  □ Right-size over-provisioned instances
  □ Enable S3 Intelligent Tiering
  □ Stop non-production resources outside business hours

Priority 2 — Committed Savings (weeks):
  □ Purchase Savings Plans for steady-state compute
  □ Reserved Instances for databases
  □ Use Spot Instances for fault-tolerant workloads

Priority 3 — Architecture Changes (months):
  □ Migrate to serverless for variable workloads
  □ Implement caching to reduce database/API calls
  □ Use CDN for static assets
  □ Consolidate microservices if over-decomposed
```

## Cloud Migration Strategies (7 Rs)

```
┌──────────────────────────────────────────────────────────┐
│ Strategy     │ Description              │ Effort │ Value │
├──────────────────────────────────────────────────────────┤
│ Retire       │ Decommission unused      │ Low    │ Low   │
│ Retain       │ Keep on-premises         │ None   │ None  │
│ Rehost       │ Lift and shift           │ Low    │ Low   │
│ Relocate     │ Move to cloud VMs        │ Low    │ Med   │
│ Repurchase   │ Move to SaaS            │ Med    │ Med   │
│ Replatform   │ Lift, tinker, and shift  │ Med    │ High  │
│ Refactor     │ Re-architect cloud-native│ High   │ High  │
└──────────────────────────────────────────────────────────┘
```

## References

- [Architecture Decisions](references/architecture-decisions.md) — Service selection decision trees, managed vs self-hosted trade-offs, multi-cloud strategy.
- [Cost Analysis](references/cost-analysis.md) — AWS Cost Explorer queries, tagging strategy, budget alerts, FinOps practices, TCO calculation.
