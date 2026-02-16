---
name: Serverless Patterns
description: This skill should be used when the user asks about "serverless", "Lambda functions", "Cloud Functions", "Cloud Run", "API Gateway", "cold start", "serverless architecture", "event-driven serverless", "fan-out", "fan-in", "Step Functions", or "serverless framework". It covers serverless architecture patterns, cold start optimization, and event-driven design.
---

# Serverless Patterns

## API Gateway + Lambda

```typescript
// Lightweight Lambda handler — minimize cold start
import { APIGatewayProxyHandlerV2 } from 'aws-lambda';

// Initialize outside handler for connection reuse
const db = new DynamoDBClient({});

export const handler: APIGatewayProxyHandlerV2 = async (event) => {
  const { httpMethod, pathParameters, body } = event;

  switch (`${httpMethod} ${event.routeKey?.split(' ')[1]}`) {
    case 'GET /users/{id}':
      return getUser(pathParameters!.id!);
    case 'POST /users':
      return createUser(JSON.parse(body!));
    default:
      return { statusCode: 404, body: 'Not Found' };
  }
};
```

## Cold Start Optimization

```
Strategies to minimize cold start latency:

1. Keep deployment packages small
   - Use esbuild/rollup to bundle (tree-shake unused code)
   - Exclude AWS SDK v3 from bundle (included in runtime)
   - Use Lambda layers for shared dependencies

2. Optimize initialization
   - Initialize SDK clients OUTSIDE handler (reused across invocations)
   - Lazy-load modules only when needed
   - Use connection pooling for database connections

3. Provisioned Concurrency
   - Pre-warm specific number of execution environments
   - Use Application Auto Scaling to adjust based on schedule
   - Cost: ~$0.015/GB-hour provisioned

4. Language choice impact on cold start:
   ┌──────────────┬───────────────┐
   │ Runtime      │ Cold Start    │
   ├──────────────┼───────────────┤
   │ Python       │ ~200-400ms    │
   │ Node.js      │ ~200-500ms    │
   │ Go           │ ~100-200ms    │
   │ Java (no SS) │ ~2-5s         │
   │ Java SnapStart│ ~200-400ms   │
   │ .NET (AOT)   │ ~200-400ms   │
   └──────────────┴───────────────┘
```

## Event-Driven Patterns

```
Fan-Out Pattern:
  SNS Topic → SQS Queue A → Lambda A (process)
            → SQS Queue B → Lambda B (notify)
            → SQS Queue C → Lambda C (analytics)

Fan-In Pattern:
  S3 Upload → Lambda (split) → SQS → Lambda Workers (parallel)
                                     → DynamoDB (results)
                                     → Step Functions (aggregate)

Saga Pattern (distributed transactions):
  Step Functions orchestrates:
    1. Reserve inventory  → (compensate: release inventory)
    2. Process payment    → (compensate: refund payment)
    3. Create shipment    → (compensate: cancel shipment)
    4. Send confirmation
  On any failure → execute compensation steps in reverse
```

## Step Functions Patterns

```json
{
  "Comment": "Parallel processing with error handling",
  "StartAt": "ProcessInParallel",
  "States": {
    "ProcessInParallel": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "ResizeImage",
          "States": {
            "ResizeImage": { "Type": "Task", "Resource": "arn:aws:lambda:...:resize", "End": true }
          }
        },
        {
          "StartAt": "ExtractMetadata",
          "States": {
            "ExtractMetadata": { "Type": "Task", "Resource": "arn:aws:lambda:...:metadata", "End": true }
          }
        }
      ],
      "Next": "SaveResults",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "HandleError"
      }]
    },
    "SaveResults": { "Type": "Task", "Resource": "arn:aws:lambda:...:save", "End": true },
    "HandleError": { "Type": "Task", "Resource": "arn:aws:lambda:...:error", "End": true }
  }
}
```

## Serverless Framework Configuration

```yaml
# serverless.yml
service: my-api
frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs20.x
  memorySize: 256
  timeout: 30
  environment:
    TABLE_NAME: !Ref MainTable
  iam:
    role:
      statements:
        - Effect: Allow
          Action: [dynamodb:GetItem, dynamodb:PutItem, dynamodb:Query]
          Resource: !GetAtt MainTable.Arn

functions:
  getUser:
    handler: src/handlers/getUser.handler
    events:
      - httpApi:
          path: /users/{id}
          method: get

  processOrder:
    handler: src/handlers/processOrder.handler
    events:
      - sqs:
          arn: !GetAtt OrderQueue.Arn
          batchSize: 10
```

## References

- [Serverless Anti-Patterns](references/serverless-antipatterns.md) — Common mistakes, Lambda limits, cost traps, when NOT to use serverless.
- [Event Source Patterns](references/event-source-patterns.md) — S3 triggers, DynamoDB Streams, Kinesis, EventBridge, scheduled events, IoT rules.
