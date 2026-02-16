# Serverless Anti-Patterns

## Common Mistakes

```
1. Lambda Monolith
   BAD:  Single Lambda with Express router handling all routes
   GOOD: One Lambda per route/function (Single Responsibility)
   WHY:  Monolith Lambda has larger package → slower cold starts,
         harder to scale, debug, and set appropriate memory/timeout

2. Synchronous Chains
   BAD:  Lambda A → calls Lambda B → calls Lambda C (sync)
   GOOD: Lambda A → SQS → Lambda B → SQS → Lambda C (async)
   WHY:  Sync chains compound latency, waste execution time waiting,
         and risk timeouts. Async decouples and improves resilience.

3. Over-Provisioned Memory
   BAD:  All Lambdas at 1024MB "just in case"
   GOOD: Use AWS Lambda Power Tuning to find optimal memory
   WHY:  Memory directly affects cost. A 256MB function costs 4x less
         than 1024MB per ms. Many functions run fine at 128-256MB.

4. Missing Dead Letter Queues
   BAD:  Lambda triggered by SQS with no DLQ
   GOOD: Every async trigger has DLQ + alarm on DLQ depth
   WHY:  Failed messages retry indefinitely, consuming capacity
         and hiding errors. DLQ captures failures for investigation.

5. Ignoring Concurrency Limits
   BAD:  No reserved concurrency, no throttling configuration
   GOOD: Set reserved concurrency per function, use SQS buffering
   WHY:  One runaway function can consume all account concurrency
         (default 1000), starving other functions.
```

## Lambda Limits to Know

```
┌──────────────────────────────────────────────────────────┐
│ Limit                        │ Value                     │
├──────────────────────────────────────────────────────────┤
│ Execution timeout            │ 15 minutes max            │
│ Memory                       │ 128 MB – 10,240 MB        │
│ Package size (zipped)        │ 50 MB (250 MB unzipped)   │
│ Temporary storage (/tmp)     │ 512 MB – 10,240 MB        │
│ Environment variables        │ 4 KB total                │
│ Concurrent executions        │ 1,000 default (adjustable)│
│ Payload (sync invoke)        │ 6 MB                      │
│ Payload (async invoke)       │ 256 KB                    │
│ Layers per function          │ 5                         │
│ Container image size         │ 10 GB                     │
└──────────────────────────────────────────────────────────┘

When you hit these limits, consider:
- Timeout → Step Functions for long-running workflows
- Package size → Lambda layers or container images
- Payload → S3 presigned URLs for large data
- Concurrency → SQS buffering with controlled batch size
```

## Cost Traps

```
1. API Gateway REST API vs HTTP API
   REST API: $3.50/million requests
   HTTP API: $1.00/million requests
   → Use HTTP API unless you need REST API features
     (usage plans, API keys, request validation, WAF)

2. Lambda@Edge vs CloudFront Functions
   Lambda@Edge: $0.60/million + compute
   CloudFront Functions: $0.10/million (1/6th the cost)
   → Use CloudFront Functions for simple header manipulation

3. DynamoDB On-Demand vs Provisioned
   On-Demand: $1.25/million WCU, $0.25/million RCU
   Provisioned: ~$0.65/million WCU (with reserved capacity)
   → On-demand for unpredictable/dev, provisioned for steady production

4. NAT Gateway for Lambda VPC access
   NAT Gateway: $0.045/hr + $0.045/GB processed
   → ~$32/month minimum + data processing charges
   → Use VPC endpoints for AWS services instead of NAT
   → Only put Lambda in VPC if it needs private resources

5. CloudWatch Logs
   Ingestion: $0.50/GB
   Storage: $0.03/GB/month
   → Set log retention policies (7-30 days)
   → Use structured logging to reduce log volume
   → Filter unnecessary logs before ingestion
```

## When NOT to Use Serverless

```
Serverless is wrong for:
  ✗ Long-running processes (>15 min) → Use ECS/EKS
  ✗ Stateful applications → Use containers with volumes
  ✗ Consistent high throughput (>1000 req/s sustained)
    → Provisioned containers may be cheaper
  ✗ WebSocket connections (long-lived) → Use ECS/App Runner
  ✗ GPU workloads → Use EC2 with GPU instances
  ✗ Sub-10ms latency requirements → Cold starts prevent this
  ✗ Large data processing per request (>10GB) → Use Batch/EMR

Serverless is great for:
  ✓ Variable traffic with idle periods
  ✓ Event-driven processing (file uploads, queue consumers)
  ✓ APIs with < 100 req/s average
  ✓ Scheduled tasks and cron jobs
  ✓ Prototypes and MVPs
  ✓ Webhook handlers
  ✓ Image/video processing pipelines
```
