# CDK Patterns

## CDK Project Structure

```
infrastructure/
├── bin/
│   └── app.ts              # Entry point, instantiate stacks
├── lib/
│   ├── constructs/          # Reusable L3 constructs
│   │   ├── api-service.ts
│   │   └── monitored-queue.ts
│   ├── stacks/
│   │   ├── network-stack.ts
│   │   ├── database-stack.ts
│   │   ├── api-stack.ts
│   │   └── monitoring-stack.ts
│   └── config.ts            # Environment configuration
├── test/
│   └── stacks/
│       ├── api-stack.test.ts
│       └── database-stack.test.ts
└── cdk.json
```

## Environment Configuration

```typescript
// lib/config.ts
interface EnvironmentConfig {
  account: string;
  region: string;
  vpcCidr: string;
  dbInstanceClass: string;
  apiDesiredCount: number;
  domainName: string;
}

const configs: Record<string, EnvironmentConfig> = {
  dev: {
    account: '111111111111',
    region: 'us-east-1',
    vpcCidr: '10.0.0.0/16',
    dbInstanceClass: 'db.t3.micro',
    apiDesiredCount: 1,
    domainName: 'dev.example.com',
  },
  production: {
    account: '222222222222',
    region: 'us-east-1',
    vpcCidr: '10.1.0.0/16',
    dbInstanceClass: 'db.r6g.large',
    apiDesiredCount: 3,
    domainName: 'api.example.com',
  },
};

// bin/app.ts
const env = process.env.CDK_ENV || 'dev';
const config = configs[env];

const networkStack = new NetworkStack(app, `${env}-network`, {
  env: { account: config.account, region: config.region },
  vpcCidr: config.vpcCidr,
});

const dbStack = new DatabaseStack(app, `${env}-database`, {
  env: { account: config.account, region: config.region },
  vpc: networkStack.vpc,
  instanceClass: config.dbInstanceClass,
});
```

## Cross-Stack References

```typescript
// network-stack.ts — exports VPC
export class NetworkStack extends Stack {
  public readonly vpc: ec2.IVpc;

  constructor(scope: Construct, id: string, props: NetworkProps) {
    super(scope, id, props);
    this.vpc = new ec2.Vpc(this, 'Vpc', {
      maxAzs: 3,
      cidr: props.vpcCidr,
    });
  }
}

// database-stack.ts — imports VPC
export class DatabaseStack extends Stack {
  public readonly dbEndpoint: string;

  constructor(scope: Construct, id: string, props: DatabaseProps) {
    super(scope, id, props);
    const db = new rds.DatabaseInstance(this, 'Database', {
      vpc: props.vpc, // From NetworkStack
      engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_16 }),
    });
    this.dbEndpoint = db.instanceEndpoint.hostname;
  }
}
```

## Custom L3 Constructs

```typescript
// constructs/monitored-queue.ts
export class MonitoredQueue extends Construct {
  public readonly queue: sqs.Queue;
  public readonly dlq: sqs.Queue;

  constructor(scope: Construct, id: string, props: { alarmEmail: string }) {
    super(scope, id);

    this.dlq = new sqs.Queue(this, 'DLQ', {
      retentionPeriod: Duration.days(14),
    });

    this.queue = new sqs.Queue(this, 'Queue', {
      deadLetterQueue: { queue: this.dlq, maxReceiveCount: 3 },
      visibilityTimeout: Duration.seconds(300),
    });

    // Alert on DLQ messages
    const alarm = new cloudwatch.Alarm(this, 'DLQAlarm', {
      metric: this.dlq.metricApproximateNumberOfMessagesVisible(),
      threshold: 1,
      evaluationPeriods: 1,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
    });

    const topic = new sns.Topic(this, 'AlarmTopic');
    topic.addSubscription(new subscriptions.EmailSubscription(props.alarmEmail));
    alarm.addAlarmAction(new cloudwatch_actions.SnsAction(topic));
  }
}
```

## Testing CDK Stacks

```typescript
import { Template, Match } from 'aws-cdk-lib/assertions';
import { App } from 'aws-cdk-lib';
import { ApiStack } from '../lib/stacks/api-stack';

describe('ApiStack', () => {
  const app = new App();
  const stack = new ApiStack(app, 'TestApi', {
    env: { account: '123456789', region: 'us-east-1' },
    desiredCount: 2,
  });
  const template = Template.fromStack(stack);

  it('should create ECS service with correct task count', () => {
    template.hasResourceProperties('AWS::ECS::Service', {
      DesiredCount: 2,
      LaunchType: 'FARGATE',
    });
  });

  it('should create ALB with health check', () => {
    template.hasResourceProperties('AWS::ElasticLoadBalancingV2::TargetGroup', {
      HealthCheckPath: '/health',
      HealthCheckProtocol: 'HTTP',
    });
  });

  it('should have auto-scaling configured', () => {
    template.hasResourceProperties('AWS::ApplicationAutoScaling::ScalableTarget', {
      MaxCapacity: Match.anyValue(),
      MinCapacity: 2,
    });
  });

  it('should not create public S3 buckets', () => {
    template.hasResourceProperties('AWS::S3::Bucket', {
      PublicAccessBlockConfiguration: {
        BlockPublicAcls: true,
        BlockPublicPolicy: true,
        IgnorePublicAcls: true,
        RestrictPublicBuckets: true,
      },
    });
  });
});

// Snapshot testing for full stack
it('should match snapshot', () => {
  expect(template.toJSON()).toMatchSnapshot();
});
```

## CDK Aspects for Policy Enforcement

```typescript
// Enforce tagging and security policies
import { Aspects, IAspect } from 'aws-cdk-lib';

class RequireEncryption implements IAspect {
  visit(node: IConstruct) {
    if (node instanceof s3.CfnBucket) {
      if (!node.bucketEncryption) {
        Annotations.of(node).addError('S3 buckets must have encryption enabled');
      }
    }
    if (node instanceof rds.CfnDBInstance) {
      if (!node.storageEncrypted) {
        Annotations.of(node).addError('RDS instances must have storage encryption');
      }
    }
  }
}

// Apply to all stacks
Aspects.of(app).add(new RequireEncryption());
```
