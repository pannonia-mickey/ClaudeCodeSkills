# Data Replication

## Aurora Global Database

```typescript
// Cross-region replication with <1s lag
const primaryCluster = new rds.DatabaseCluster(this, 'Primary', {
  engine: rds.DatabaseClusterEngine.auroraPostgres({
    version: rds.AuroraPostgresEngineVersion.VER_16_1,
  }),
  instances: 2,
  vpc: primaryVpc,
  storageEncrypted: true,
});

// Secondary region (in a separate stack/account)
// Aurora Global Database replicates at the storage layer
// RPO: typically < 1 second
// Failover: promote secondary to primary (RTO: ~1 minute)
// Read from secondary: yes (read-only endpoint)

// Failover procedure:
// 1. Verify replication lag is minimal
// 2. Remove secondary cluster from global database
// 3. Promote secondary to standalone primary
// 4. Update application connection strings
// 5. Repoint DNS (Route 53 failover)
```

## DynamoDB Global Tables

```typescript
// Multi-region active-active with DynamoDB
const table = new dynamodb.Table(this, 'GlobalTable', {
  partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  replicationRegions: ['eu-west-1', 'ap-southeast-1'],
  // Global tables use last-writer-wins for conflict resolution
  // Items replicated within ~1 second
});

// Best practices for Global Tables:
// - Use UUID-based keys to avoid write conflicts
// - Design for eventual consistency
// - Use conditional writes for conflict-sensitive operations
// - Region-specific attributes: include region in SK
//   SK: "ORDER#us-east-1#12345" (avoid conflicts across regions)
```

## S3 Cross-Region Replication

```typescript
const sourceBucket = new s3.Bucket(this, 'Source', {
  versioned: true, // Required for replication
  encryption: s3.BucketEncryption.S3_MANAGED,
});

const destBucket = new s3.Bucket(destStack, 'Destination', {
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
});

// Replication rule
new s3.CfnBucket(this, 'SourceWithReplication', {
  replicationConfiguration: {
    role: replicationRole.roleArn,
    rules: [{
      status: 'Enabled',
      destination: {
        bucket: destBucket.bucketArn,
        storageClass: 'STANDARD_IA', // Save costs on replica
        replicationTime: {
          status: 'Enabled',
          time: { minutes: 15 }, // S3 RTC: 99.99% within 15 min
        },
        metrics: {
          status: 'Enabled',
          eventThreshold: { minutes: 15 },
        },
      },
      filter: { prefix: 'critical/' }, // Replicate specific prefix
    }],
  },
});
```

## ElastiCache Global Datastore

```
Primary Region (US-East-1):
┌─────────────────────────────┐
│ Redis Cluster (Read/Write)  │
│ 3 nodes, r6g.large          │
└─────────────┬───────────────┘
              │ Async replication
              │ (~1s lag)
Secondary Region (EU-West-1):
┌─────────────────────────────┐
│ Redis Cluster (Read-Only)   │
│ 3 nodes, r6g.large          │
└─────────────────────────────┘

Failover: Promote secondary to primary
- Automatic failover: not supported (manual promotion)
- RTO: ~1-2 minutes for promotion
- RPO: ~1 second (async replication lag)

Use cases:
- Session data replication for multi-region apps
- Cache warming in secondary region
- Read-heavy workloads served locally
```

## Consistency Models

```
┌────────────────────────────────────────────────────────────┐
│ Model              │ Guarantee          │ AWS Service      │
├────────────────────────────────────────────────────────────┤
│ Strong Consistency │ Read reflects      │ DynamoDB (per-   │
│                    │ latest write       │ request option)  │
├────────────────────────────────────────────────────────────┤
│ Eventual           │ Reads converge     │ S3, DynamoDB     │
│ Consistency        │ to latest write    │ Global Tables    │
├────────────────────────────────────────────────────────────┤
│ Read-after-write   │ Own writes visible │ DynamoDB (same   │
│ Consistency        │ immediately        │ region), Aurora   │
├────────────────────────────────────────────────────────────┤
│ Session            │ Consistency within │ ElastiCache      │
│ Consistency        │ same session       │ (sticky sessions)│
└────────────────────────────────────────────────────────────┘

Conflict Resolution Strategies:
1. Last-Writer-Wins (LWW): Simplest, used by DynamoDB Global Tables
   - Timestamp-based, higher timestamp wins
   - Risk: concurrent writes → data loss

2. Application-Level Resolution:
   - Merge changes (CRDTs — Conflict-free Replicated Data Types)
   - Queue conflicts for human review
   - Domain-specific rules (e.g., max value wins for counters)

3. Conditional Writes:
   - DynamoDB: ConditionExpression to prevent stale overwrites
   - Aurora: SELECT FOR UPDATE in transactions
```

## Replication Monitoring

```
Key metrics to monitor:
- Replication lag (seconds behind primary)
- Replication throughput (bytes/second)
- Replication errors/failures
- Cross-region data transfer costs

Alerting thresholds:
- Aurora: ReplicaLag > 100ms for > 5 minutes
- DynamoDB: ReplicationLatency > 5s for > 10 minutes
- S3 RTC: ObjectsPendingReplication > 0 for > 30 minutes
- ElastiCache: ReplicationLag > 2s for > 5 minutes
```
