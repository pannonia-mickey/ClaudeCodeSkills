# Event Source Patterns

## S3 Event Triggers

```typescript
// Process uploaded files
const bucket = new s3.Bucket(this, 'Uploads', {
  eventBridgeEnabled: true, // Prefer EventBridge over S3 notifications
});

// EventBridge rule for S3 events (more flexible filtering)
new events.Rule(this, 'ImageUploadRule', {
  eventPattern: {
    source: ['aws.s3'],
    detailType: ['Object Created'],
    detail: {
      bucket: { name: [bucket.bucketName] },
      object: { key: [{ suffix: '.jpg' }, { suffix: '.png' }] },
    },
  },
  targets: [new targets.LambdaFunction(imageProcessorFn)],
});

// Lambda handler for S3 events
export const handler = async (event: S3Event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key);
    const size = record.s3.object.size;

    // Download, process, upload
    const object = await s3.getObject({ Bucket: bucket, Key: key });
    const processed = await processImage(object.Body);
    await s3.putObject({
      Bucket: bucket,
      Key: `processed/${key}`,
      Body: processed,
    });
  }
};
```

## DynamoDB Streams

```typescript
// React to data changes in DynamoDB
const table = new dynamodb.Table(this, 'Orders', {
  partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
});

// Lambda processes stream events
const streamHandler = new lambda.Function(this, 'StreamProcessor', {
  handler: 'stream.handler',
  runtime: lambda.Runtime.NODEJS_20_X,
  events: [
    new DynamoEventSource(table, {
      startingPosition: lambda.StartingPosition.LATEST,
      batchSize: 100,
      maxBatchingWindow: Duration.seconds(5),
      bisectBatchOnError: true,
      retryAttempts: 3,
      filters: [
        lambda.FilterCriteria.filter({
          eventName: lambda.FilterRule.isEqual('INSERT'),
          dynamodb: {
            NewImage: {
              status: { S: lambda.FilterRule.isEqual('confirmed') },
            },
          },
        }),
      ],
    }),
  ],
});

// Handler
export const handler = async (event: DynamoDBStreamEvent) => {
  for (const record of event.Records) {
    if (record.eventName === 'INSERT') {
      const newImage = unmarshall(record.dynamodb!.NewImage!);
      await notifyUser(newImage.userId, newImage.orderId);
    }
  }
};
```

## Kinesis Data Streams

```typescript
// Real-time data processing
const stream = new kinesis.Stream(this, 'ClickStream', {
  shardCount: 2,
  retentionPeriod: Duration.hours(24),
});

new lambda.Function(this, 'ClickProcessor', {
  events: [
    new KinesisEventSource(stream, {
      startingPosition: lambda.StartingPosition.LATEST,
      batchSize: 100,
      maxBatchingWindow: Duration.seconds(10),
      parallelizationFactor: 10, // Process up to 10 batches per shard
      tumblingWindow: Duration.minutes(1), // Aggregate per window
    }),
  ],
});

// Producer
import { KinesisClient, PutRecordCommand } from '@aws-sdk/client-kinesis';

const kinesis = new KinesisClient({});
await kinesis.send(new PutRecordCommand({
  StreamName: 'ClickStream',
  Data: Buffer.from(JSON.stringify({ userId: 'u1', page: '/products', ts: Date.now() })),
  PartitionKey: 'u1', // Route all user events to same shard for ordering
}));
```

## EventBridge Scheduled Events

```typescript
// Cron jobs with EventBridge
new events.Rule(this, 'DailyCleanup', {
  schedule: events.Schedule.cron({
    minute: '0',
    hour: '2',
    weekDay: 'MON-FRI',
  }),
  targets: [new targets.LambdaFunction(cleanupFn)],
});

// Rate-based schedule
new events.Rule(this, 'HealthCheck', {
  schedule: events.Schedule.rate(Duration.minutes(5)),
  targets: [new targets.LambdaFunction(healthCheckFn)],
});
```

## EventBridge Pipes

```typescript
// Direct integration: SQS → transform → Step Functions
new pipes.CfnPipe(this, 'OrderPipe', {
  source: orderQueue.queueArn,
  sourceParameters: {
    sqsQueueParameters: { batchSize: 10 },
    filterCriteria: {
      filters: [{ pattern: '{"body": {"type": ["order.created"]}}' }],
    },
  },
  enrichment: enrichmentFn.functionArn,
  target: orderWorkflow.stateMachineArn,
  targetParameters: {
    stepFunctionStateMachineParameters: {
      invocationType: 'FIRE_AND_FORGET',
    },
  },
  roleArn: pipeRole.roleArn,
});
```

## Multi-Source Event Handling

```typescript
// Single Lambda handling multiple event sources
export const handler = async (event: unknown) => {
  // Detect event source
  if ('Records' in event && event.Records[0]?.eventSource === 'aws:sqs') {
    return handleSQS(event as SQSEvent);
  }
  if ('Records' in event && event.Records[0]?.eventSource === 'aws:s3') {
    return handleS3(event as S3Event);
  }
  if ('detail-type' in event) {
    return handleEventBridge(event as EventBridgeEvent);
  }
  if ('httpMethod' in event || 'routeKey' in event) {
    return handleAPI(event as APIGatewayProxyEventV2);
  }

  throw new Error(`Unknown event source: ${JSON.stringify(event).slice(0, 200)}`);
};
```
