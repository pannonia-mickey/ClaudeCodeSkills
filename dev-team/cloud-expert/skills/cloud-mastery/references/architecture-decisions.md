# Architecture Decisions

## Service Selection Decision Tree

```
Need a database?
├── Relational data with complex queries?
│   ├── High availability + auto-scaling? → Aurora Serverless v2
│   ├── Standard RDBMS? → RDS PostgreSQL / MySQL
│   └── Global distribution? → Aurora Global / CockroachDB
├── Key-value with millisecond latency?
│   ├── Simple key-value? → DynamoDB / ElastiCache
│   └── Rich data structures? → Redis (ElastiCache)
├── Document store?
│   ├── AWS native? → DynamoDB (single-table)
│   └── MongoDB API? → DocumentDB / MongoDB Atlas
├── Time-series data? → Timestream / InfluxDB
├── Graph relationships? → Neptune
└── Full-text search? → OpenSearch

Need compute?
├── Stateless HTTP handlers? → Lambda / Cloud Functions
├── Long-running containers? → ECS Fargate / Cloud Run
├── Kubernetes required? → EKS / GKE / AKS
├── Batch processing? → AWS Batch / Cloud Run Jobs
└── GPU workloads? → EC2 P-series / GCP A2/A3

Need messaging?
├── Point-to-point queue? → SQS / Cloud Tasks
├── Pub/sub fan-out? → SNS + SQS / Pub/Sub
├── Event routing + filtering? → EventBridge
├── Streaming (ordered, replay)? → Kinesis / Kafka (MSK)
└── Workflow orchestration? → Step Functions / Workflows
```

## Managed vs Self-Hosted

```
┌──────────────────────────────────────────────────────────────┐
│ Service       │ Managed                │ Self-Hosted         │
├──────────────────────────────────────────────────────────────┤
│ PostgreSQL    │ RDS/Aurora             │ EC2 + Patroni       │
│ When managed: │ 95% of cases. Backups, │ Custom extensions,  │
│               │ failover, patching.    │ extreme tuning needs│
├──────────────────────────────────────────────────────────────┤
│ Redis         │ ElastiCache            │ EC2 + Redis         │
│ When managed: │ 99% of cases.          │ Custom modules only │
├──────────────────────────────────────────────────────────────┤
│ Kafka         │ MSK / Confluent Cloud  │ EC2 + Kafka         │
│ When managed: │ Unless cost-prohibitive│ High-volume, need   │
│               │ at scale.              │ fine-grained control│
├──────────────────────────────────────────────────────────────┤
│ Kubernetes    │ EKS/GKE/AKS           │ kubeadm/k3s         │
│ When managed: │ Always for production. │ Edge/IoT, air-gap   │
└──────────────────────────────────────────────────────────────┘

Rule of thumb: Use managed unless you have a specific, technical reason
not to. The operational overhead of self-hosting is almost never worth it.
```

## Multi-Cloud Strategy

```
When multi-cloud makes sense:
  ✓ Regulatory requirement (data sovereignty, vendor lock-in clauses)
  ✓ Best-of-breed services (GCP BigQuery + AWS Lambda)
  ✓ Acquisition (merging companies on different clouds)
  ✓ Negotiation leverage for large contracts

When multi-cloud is wasteful:
  ✗ "Just in case" without specific driver
  ✗ Small team (< 20 engineers) — operational overhead too high
  ✗ Tightly coupled services that need low latency
  ✗ No expertise in second cloud

Practical approach:
  - Primary cloud for 90% of workloads
  - Use cloud-agnostic for stateless compute (containers, K8s)
  - Accept vendor lock-in for managed services that provide 10x value
  - Use Terraform/Pulumi for IaC portability
```

## Architecture Decision Record (ADR) Template

```markdown
# ADR-001: Use Aurora Serverless v2 for Primary Database

## Status: Accepted

## Context
Our application needs a relational database that can handle variable traffic
patterns (10 req/s baseline, 1000 req/s during sales events). We need
PostgreSQL compatibility for our existing ORM and queries.

## Decision
Use Aurora Serverless v2 with PostgreSQL 15 compatibility.

## Consequences
### Positive
- Auto-scales ACUs (0.5 to 128) matching traffic patterns
- No instance management or capacity planning
- Pay-per-use during low-traffic periods
- Built-in high availability (multi-AZ)

### Negative
- Higher per-ACU cost vs provisioned Aurora for sustained loads
- Minimum 0.5 ACU charge even when idle
- Limited to Aurora-supported PostgreSQL extensions
- Cold start latency when scaling from minimum ACUs

### Alternatives Considered
1. RDS PostgreSQL — rejected: requires manual scaling, over-provision for peak
2. DynamoDB — rejected: relational queries needed, team expertise in PostgreSQL
3. Self-managed PostgreSQL on EC2 — rejected: operational overhead
```
