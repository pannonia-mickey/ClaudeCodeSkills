---
name: AWS Services
description: This skill should be used when the user asks about "AWS Lambda", "ECS", "EKS", "S3", "RDS", "Aurora", "DynamoDB", "SQS", "SNS", "CloudFront", "API Gateway", "Cognito", "IAM", "CDK", "CloudFormation", or any specific AWS service configuration. It covers core AWS service patterns and CDK/CloudFormation.
---

# AWS Services

## Lambda Function Patterns

```typescript
// Lambda handler with Powertools
import { Logger } from '@aws-lambda-powertools/logger';
import { Tracer } from '@aws-lambda-powertools/tracer';
import { Metrics, MetricUnit } from '@aws-lambda-powertools/metrics';
import middy from '@middy/core';
import httpJsonBodyParser from '@middy/http-json-body-parser';
import httpErrorHandler from '@middy/http-error-handler';

const logger = new Logger({ serviceName: 'order-service' });
const tracer = new Tracer({ serviceName: 'order-service' });
const metrics = new Metrics({ namespace: 'OrderService' });

const handler = async (event: APIGatewayProxyEventV2) => {
  logger.info('Processing order', { orderId: event.pathParameters?.id });

  const order = await orderService.create(JSON.parse(event.body!));
  metrics.addMetric('OrderCreated', MetricUnit.Count, 1);

  return { statusCode: 201, body: JSON.stringify(order) };
};

export const main = middy(handler)
  .use(httpJsonBodyParser())
  .use(httpErrorHandler());
```

## ECS Fargate Service

```yaml
# AWS CDK — ECS Fargate with ALB
# cdk/lib/api-stack.ts
const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

const service = new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'Api', {
  cluster,
  cpu: 512,
  memoryLimitMiB: 1024,
  desiredCount: 2,
  taskImageOptions: {
    image: ecs.ContainerImage.fromEcrRepository(repo, tag),
    containerPort: 3000,
    environment: {
      NODE_ENV: 'production',
      DATABASE_URL: databaseUrl.stringValue,
    },
    logDriver: ecs.LogDrivers.awsLogs({ streamPrefix: 'api' }),
  },
  healthCheckGracePeriod: Duration.seconds(60),
});

service.targetGroup.configureHealthCheck({
  path: '/health',
  interval: Duration.seconds(30),
  healthyThresholdCount: 2,
});

const scaling = service.service.autoScaleTaskCount({ maxCapacity: 10 });
scaling.scaleOnCpuUtilization('CpuScaling', { targetUtilizationPercent: 70 });
scaling.scaleOnRequestCount('RequestScaling', {
  targetGroup: service.targetGroup,
  requestsPerTarget: 1000,
});
```

## DynamoDB Single-Table Design

```typescript
// Single-table design with composite keys
// PK: USER#<userId>     SK: PROFILE
// PK: USER#<userId>     SK: ORDER#<orderId>
// PK: ORDER#<orderId>   SK: ITEM#<itemId>

const table = new dynamodb.Table(this, 'MainTable', {
  partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  pointInTimeRecovery: true,
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
});

// GSI for access patterns
table.addGlobalSecondaryIndex({
  indexName: 'GSI1',
  partitionKey: { name: 'GSI1PK', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'GSI1SK', type: dynamodb.AttributeType.STRING },
});

// Query: Get all orders for a user
const result = await dynamodb.query({
  TableName: 'MainTable',
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': `USER#${userId}`,
    ':sk': 'ORDER#',
  },
});
```

## S3 + CloudFront

```typescript
// Static site hosting with CloudFront
const bucket = new s3.Bucket(this, 'WebBucket', {
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  encryption: s3.BucketEncryption.S3_MANAGED,
  lifecycleRules: [
    { id: 'intelligent-tiering', transitions: [{ storageClass: s3.StorageClass.INTELLIGENT_TIERING, transitionAfter: Duration.days(30) }] },
  ],
});

const distribution = new cloudfront.Distribution(this, 'CDN', {
  defaultBehavior: {
    origin: new origins.S3Origin(bucket),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
  },
  defaultRootObject: 'index.html',
  errorResponses: [
    { httpStatus: 404, responseHttpStatus: 200, responsePagePath: '/index.html' }, // SPA fallback
  ],
});
```

## References

- [AWS Patterns](references/aws-patterns.md) — SQS/SNS event patterns, Step Functions workflows, EventBridge rules, Cognito auth flows.
- [CDK Patterns](references/cdk-patterns.md) — CDK constructs, cross-stack references, environment configuration, testing CDK stacks.
