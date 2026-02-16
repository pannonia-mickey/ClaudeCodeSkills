---
name: Cloud Security
description: This skill should be used when the user asks about "IAM policies", "VPC security", "cloud encryption", "security groups", "WAF", "DDoS protection", "cloud compliance", "network security", "cloud IAM", or "cloud security best practices". It covers IAM, networking, encryption, compliance, and cloud-specific security patterns.
---

# Cloud Security

## IAM Best Practices

```json
// Least-privilege IAM policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/uploads/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/team": "backend"
        }
      }
    }
  ]
}
```

```typescript
// CDK — role with minimal permissions
const lambdaRole = new iam.Role(this, 'LambdaRole', {
  assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
  managedPolicies: [
    iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'),
  ],
});

// Grant specific access (prefer grant* methods over inline policies)
table.grantReadWriteData(lambdaRole);
bucket.grantRead(lambdaRole);
queue.grantSendMessages(lambdaRole);

// IAM Access Analyzer — detect unused permissions
// aws accessanalyzer create-analyzer --analyzer-name my-analyzer --type ACCOUNT
```

## VPC Security Architecture

```
┌─────────────────────────────────────────────────────────┐
│                         VPC                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Public Subnets                       │   │
│  │   ALB ──── NAT Gateway ──── Bastion (optional)   │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Private Subnets (App)                │   │
│  │   ECS Tasks / EC2 Instances / Lambda              │   │
│  │   SG: Allow ALB on port 3000                      │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Private Subnets (Data)               │   │
│  │   RDS / ElastiCache / OpenSearch                  │   │
│  │   SG: Allow App subnets on DB ports only          │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  VPC Endpoints: S3, DynamoDB, SQS, SNS, Secrets Manager │
│  (Traffic stays in AWS network, avoids NAT costs)        │
└─────────────────────────────────────────────────────────┘
```

## Encryption

```typescript
// Encryption at rest — KMS
const key = new kms.Key(this, 'AppKey', {
  enableKeyRotation: true, // Automatic annual rotation
  alias: 'alias/my-app-key',
  policy: new iam.PolicyDocument({
    statements: [
      new iam.PolicyStatement({
        principals: [new iam.AccountRootPrincipal()],
        actions: ['kms:*'],
        resources: ['*'],
      }),
    ],
  }),
});

// Use KMS for RDS, S3, EBS, SQS
const db = new rds.DatabaseInstance(this, 'DB', {
  storageEncrypted: true,
  storageEncryptionKey: key,
});

// Encryption in transit — enforce HTTPS
const bucket = new s3.Bucket(this, 'SecureBucket', {
  encryption: s3.BucketEncryption.KMS,
  encryptionKey: key,
  enforceSSL: true, // Deny HTTP requests
});
```

## WAF Configuration

```typescript
// AWS WAF with managed rules
const webAcl = new wafv2.CfnWebACL(this, 'WebACL', {
  defaultAction: { allow: {} },
  scope: 'REGIONAL',
  visibilityConfig: {
    sampledRequestsEnabled: true,
    cloudWatchMetricsEnabled: true,
    metricName: 'WebACL',
  },
  rules: [
    {
      name: 'AWSManagedRulesCommonRuleSet',
      priority: 1,
      overrideAction: { none: {} },
      statement: {
        managedRuleGroupStatement: {
          vendorName: 'AWS',
          name: 'AWSManagedRulesCommonRuleSet',
        },
      },
      visibilityConfig: { sampledRequestsEnabled: true, cloudWatchMetricsEnabled: true, metricName: 'Common' },
    },
    {
      name: 'SQLInjection',
      priority: 2,
      overrideAction: { none: {} },
      statement: {
        managedRuleGroupStatement: {
          vendorName: 'AWS',
          name: 'AWSManagedRulesSQLiRuleSet',
        },
      },
      visibilityConfig: { sampledRequestsEnabled: true, cloudWatchMetricsEnabled: true, metricName: 'SQLi' },
    },
    {
      name: 'RateLimit',
      priority: 3,
      action: { block: {} },
      statement: {
        rateBasedStatement: {
          limit: 2000,
          aggregateKeyType: 'IP',
        },
      },
      visibilityConfig: { sampledRequestsEnabled: true, cloudWatchMetricsEnabled: true, metricName: 'RateLimit' },
    },
  ],
});
```

## References

- [IAM Patterns](references/iam-patterns.md) — Cross-account access, service-linked roles, permission boundaries, ABAC, identity federation.
- [Network Security](references/network-security.md) — Security groups vs NACLs, VPC endpoints, PrivateLink, Transit Gateway, GuardDuty, Inspector.
