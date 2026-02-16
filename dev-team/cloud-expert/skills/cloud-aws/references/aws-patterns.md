# AWS Patterns

## SQS/SNS Event Patterns

```typescript
// Fan-out: SNS → Multiple SQS queues
const topic = new sns.Topic(this, 'OrderEvents');

const inventoryQueue = new sqs.Queue(this, 'InventoryQueue', {
  deadLetterQueue: { queue: inventoryDLQ, maxReceiveCount: 3 },
  visibilityTimeout: Duration.seconds(300),
});

const notificationQueue = new sqs.Queue(this, 'NotificationQueue', {
  deadLetterQueue: { queue: notificationDLQ, maxReceiveCount: 3 },
});

topic.addSubscription(new subscriptions.SqsSubscription(inventoryQueue, {
  filterPolicy: {
    eventType: sns.SubscriptionFilter.stringFilter({
      allowlist: ['order.created', 'order.cancelled'],
    }),
  },
}));

topic.addSubscription(new subscriptions.SqsSubscription(notificationQueue));

// Lambda consumer with batch processing
const handler = new lambda.Function(this, 'ProcessOrders', {
  handler: 'index.handler',
  runtime: lambda.Runtime.NODEJS_20_X,
  events: [
    new SqsEventSource(inventoryQueue, {
      batchSize: 10,
      maxBatchingWindow: Duration.seconds(5),
      reportBatchItemFailures: true, // Partial batch failure
    }),
  ],
});
```

## Step Functions Workflow

```typescript
// Order processing workflow
const validateOrder = new tasks.LambdaInvoke(this, 'Validate', {
  lambdaFunction: validateFn,
  outputPath: '$.Payload',
});

const processPayment = new tasks.LambdaInvoke(this, 'Payment', {
  lambdaFunction: paymentFn,
  outputPath: '$.Payload',
  retryOnServiceExceptions: true,
});

const sendNotification = new tasks.SnsPublish(this, 'Notify', {
  topic: notificationTopic,
  message: sfn.TaskInput.fromJsonPathAt('$.orderId'),
});

const handleFailure = new tasks.SnsPublish(this, 'AlertFailure', {
  topic: alertTopic,
  message: sfn.TaskInput.fromJsonPathAt('$.error'),
});

const definition = validateOrder
  .next(new sfn.Choice(this, 'IsValid?')
    .when(sfn.Condition.booleanEquals('$.valid', true),
      processPayment
        .addCatch(handleFailure, { errors: ['PaymentFailed'] })
        .next(sendNotification))
    .otherwise(handleFailure));

new sfn.StateMachine(this, 'OrderWorkflow', {
  definitionBody: sfn.DefinitionBody.fromChainable(definition),
  timeout: Duration.minutes(5),
  tracingEnabled: true,
});
```

## EventBridge Rules

```typescript
// Event-driven architecture with EventBridge
const bus = new events.EventBus(this, 'AppEventBus');

// Rule: Route order events to processing Lambda
new events.Rule(this, 'OrderCreatedRule', {
  eventBus: bus,
  eventPattern: {
    source: ['com.myapp.orders'],
    detailType: ['OrderCreated'],
    detail: {
      total: [{ numeric: ['>=', 100] }], // Only high-value orders
    },
  },
  targets: [new targets.LambdaFunction(highValueOrderFn)],
});

// Publish events
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const eb = new EventBridgeClient({});
await eb.send(new PutEventsCommand({
  Entries: [{
    Source: 'com.myapp.orders',
    DetailType: 'OrderCreated',
    Detail: JSON.stringify({ orderId: '123', total: 250, userId: 'u1' }),
    EventBusName: 'AppEventBus',
  }],
}));
```

## Cognito Auth Flow

```typescript
// Cognito User Pool with custom attributes
const userPool = new cognito.UserPool(this, 'UserPool', {
  selfSignUpEnabled: true,
  signInAliases: { email: true },
  standardAttributes: {
    email: { required: true, mutable: false },
    fullname: { required: true, mutable: true },
  },
  passwordPolicy: {
    minLength: 12,
    requireUppercase: true,
    requireDigits: true,
    requireSymbols: true,
  },
  accountRecovery: cognito.AccountRecovery.EMAIL_ONLY,
  mfa: cognito.Mfa.OPTIONAL,
  mfaSecondFactor: { sms: false, otp: true },
});

const client = userPool.addClient('WebApp', {
  oAuth: {
    flows: { authorizationCodeGrant: true },
    scopes: [cognito.OAuthScope.OPENID, cognito.OAuthScope.EMAIL, cognito.OAuthScope.PROFILE],
    callbackUrls: ['https://app.example.com/callback'],
  },
  authFlows: { userSrp: true },
});

// API Gateway with Cognito authorizer
const authorizer = new apigateway.CognitoUserPoolsAuthorizer(this, 'Authorizer', {
  cognitoUserPools: [userPool],
});
```

## SES Email

```typescript
// Transactional email with SES
import { SESv2Client, SendEmailCommand } from '@aws-sdk/client-sesv2';

const ses = new SESv2Client({});
await ses.send(new SendEmailCommand({
  FromEmailAddress: 'noreply@example.com',
  Destination: { ToAddresses: [user.email] },
  Content: {
    Template: {
      TemplateName: 'OrderConfirmation',
      TemplateData: JSON.stringify({ name: user.name, orderId: order.id }),
    },
  },
}));
```
