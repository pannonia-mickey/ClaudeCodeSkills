# Network Security

## Security Groups vs NACLs

```
┌──────────────────────────────────────────────────────────┐
│ Feature           │ Security Group    │ NACL              │
├──────────────────────────────────────────────────────────┤
│ Level             │ Instance/ENI      │ Subnet            │
│ State             │ Stateful          │ Stateless         │
│ Rules             │ Allow only        │ Allow + Deny      │
│ Evaluation        │ All rules checked │ Rules in order    │
│ Default           │ Deny all inbound  │ Allow all         │
│ Use for           │ App-level access  │ Subnet-level deny │
└──────────────────────────────────────────────────────────┘
```

```typescript
// Security group best practices
const appSg = new ec2.SecurityGroup(this, 'AppSG', {
  vpc,
  description: 'Allow ALB to app containers on port 3000',
  allowAllOutbound: false, // Explicit outbound rules
});

// Only allow traffic from ALB
appSg.addIngressRule(
  ec2.Peer.securityGroupId(albSg.securityGroupId),
  ec2.Port.tcp(3000),
  'ALB to app'
);

// Only allow outbound to specific destinations
appSg.addEgressRule(
  ec2.Peer.securityGroupId(dbSg.securityGroupId),
  ec2.Port.tcp(5432),
  'App to PostgreSQL'
);
appSg.addEgressRule(
  ec2.Peer.securityGroupId(redisSg.securityGroupId),
  ec2.Port.tcp(6379),
  'App to Redis'
);
appSg.addEgressRule(
  ec2.Peer.prefixList(s3PrefixList.prefixListId),
  ec2.Port.tcp(443),
  'App to S3 via VPC endpoint'
);
```

## VPC Endpoints

```typescript
// Gateway endpoints (free) — S3, DynamoDB
vpc.addGatewayEndpoint('S3Endpoint', {
  service: ec2.GatewayVpcEndpointAwsService.S3,
});

vpc.addGatewayEndpoint('DynamoEndpoint', {
  service: ec2.GatewayVpcEndpointAwsService.DYNAMODB,
});

// Interface endpoints (PrivateLink) — other AWS services
// Avoids NAT Gateway costs for AWS API calls
const endpoints = [
  { id: 'SQS',     service: ec2.InterfaceVpcEndpointAwsService.SQS },
  { id: 'SNS',     service: ec2.InterfaceVpcEndpointAwsService.SNS },
  { id: 'Secrets', service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER },
  { id: 'ECR',     service: ec2.InterfaceVpcEndpointAwsService.ECR },
  { id: 'ECRDKR',  service: ec2.InterfaceVpcEndpointAwsService.ECR_DOCKER },
  { id: 'Logs',    service: ec2.InterfaceVpcEndpointAwsService.CLOUDWATCH_LOGS },
];

for (const ep of endpoints) {
  vpc.addInterfaceEndpoint(ep.id, {
    service: ep.service,
    privateDnsEnabled: true,
    securityGroups: [endpointSg],
    subnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  });
}
```

## AWS PrivateLink (Service-to-Service)

```typescript
// Expose internal service to other VPCs/accounts
// Provider side — NLB backed service
const nlb = new elbv2.NetworkLoadBalancer(this, 'NLB', {
  vpc,
  internetFacing: false,
});

const endpointService = new ec2.VpcEndpointService(this, 'EndpointService', {
  vpcEndpointServiceLoadBalancers: [nlb],
  acceptanceRequired: true, // Manual approval
  allowedPrincipals: [
    new iam.ArnPrincipal('arn:aws:iam::111111111111:root'),
  ],
});

// Consumer side — connect to service
const endpoint = new ec2.InterfaceVpcEndpoint(this, 'ServiceEndpoint', {
  vpc: consumerVpc,
  service: new ec2.InterfaceVpcEndpointService(
    endpointService.vpcEndpointServiceName
  ),
});
```

## GuardDuty Threat Detection

```typescript
// Enable GuardDuty for threat detection
new guardduty.CfnDetector(this, 'GuardDuty', {
  enable: true,
  dataSources: {
    s3Logs: { enable: true },
    kubernetes: { auditLogs: { enable: true } },
    malwareProtection: {
      scanEc2InstanceWithFindings: {
        ebsVolumes: true,
      },
    },
  },
  findingPublishingFrequency: 'FIFTEEN_MINUTES',
});

// Alert on high-severity findings
const findingsRule = new events.Rule(this, 'GuardDutyFindings', {
  eventPattern: {
    source: ['aws.guardduty'],
    detailType: ['GuardDuty Finding'],
    detail: {
      severity: [{ numeric: ['>=', 7] }], // High severity
    },
  },
  targets: [
    new targets.SnsTopic(securityAlertTopic),
    new targets.LambdaFunction(securityResponseFn),
  ],
});
```

## VPC Flow Logs

```typescript
// Enable VPC Flow Logs for network monitoring
const flowLog = new ec2.FlowLog(this, 'FlowLog', {
  resourceType: ec2.FlowLogResourceType.fromVpc(vpc),
  destination: ec2.FlowLogDestination.toCloudWatchLogs(
    new logs.LogGroup(this, 'FlowLogGroup', {
      retention: logs.RetentionDays.THIRTY_DAYS,
    })
  ),
  trafficType: ec2.FlowLogTrafficType.REJECT, // Log rejected traffic
  maxAggregationInterval: FlowLogMaxAggregationInterval.ONE_MINUTE,
});

// Analyze rejected traffic patterns:
// CloudWatch Insights query:
// fields @timestamp, srcAddr, dstAddr, dstPort, action
// | filter action = "REJECT"
// | stats count(*) as rejectCount by srcAddr, dstPort
// | sort rejectCount desc
```

## Transit Gateway (Multi-VPC)

```
┌──────────────────────────────────────────────────────┐
│                  Transit Gateway                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │ Prod VPC │ │ Dev VPC  │ │ Shared   │            │
│  │ 10.1.0.0 │ │ 10.2.0.0 │ │ Services │            │
│  └──────────┘ └──────────┘ │ 10.0.0.0 │            │
│                             └──────────┘            │
│  Route Tables:                                       │
│  - Prod → Shared (monitoring, logging)               │
│  - Dev  → Shared (monitoring, logging)               │
│  - Prod ✗ Dev (isolated)                             │
│  - On-premises via VPN/Direct Connect                │
└──────────────────────────────────────────────────────┘
```
