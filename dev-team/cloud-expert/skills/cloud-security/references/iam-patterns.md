# IAM Patterns

## Cross-Account Access

```json
// Account A (resource account) — trust policy on role
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:role/deployment-role"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-external-id"
      }
    }
  }]
}

// Account B (caller) — assume role
// aws sts assume-role \
//   --role-arn arn:aws:iam::222222222222:role/cross-account-deploy \
//   --role-session-name deploy-session \
//   --external-id unique-external-id
```

```typescript
// CDK — cross-account role
const crossAccountRole = new iam.Role(this, 'CrossAccountRole', {
  assumedBy: new iam.AccountPrincipal('111111111111'),
  externalIds: ['unique-external-id'],
  maxSessionDuration: Duration.hours(1),
});

crossAccountRole.addToPolicy(new iam.PolicyStatement({
  actions: ['s3:GetObject', 's3:PutObject'],
  resources: ['arn:aws:s3:::shared-bucket/*'],
}));
```

## Permission Boundaries

```json
// Limit maximum permissions — even if role policy grants more
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServices",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "sqs:*",
        "logs:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDangerousActions",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:CreateRole",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RestrictRegion",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "sts:*",
        "cloudfront:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
        }
      }
    }
  ]
}
```

## Attribute-Based Access Control (ABAC)

```json
// Tag-based access — scale permissions without policy updates
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:StartInstances",
      "ec2:StopInstances"
    ],
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
      }
    }
  }]
}

// User tagged with team=backend can only manage
// EC2 instances also tagged with team=backend
// No policy updates needed when adding new resources
```

## Identity Federation

```typescript
// OIDC identity provider for GitHub Actions
const githubProvider = new iam.OpenIdConnectProvider(this, 'GitHub', {
  url: 'https://token.actions.githubusercontent.com',
  clientIds: ['sts.amazonaws.com'],
  thumbprints: ['6938fd4d98bab03faadb97b34396831e3780aea1'],
});

// Role that GitHub Actions can assume
const deployRole = new iam.Role(this, 'GitHubDeploy', {
  assumedBy: new iam.FederatedPrincipal(
    githubProvider.openIdConnectProviderArn,
    {
      StringEquals: {
        'token.actions.githubusercontent.com:aud': 'sts.amazonaws.com',
      },
      StringLike: {
        'token.actions.githubusercontent.com:sub': 'repo:myorg/myrepo:ref:refs/heads/main',
      },
    },
    'sts:AssumeRoleWithWebIdentity'
  ),
});

// GitHub Actions workflow
// - uses: aws-actions/configure-aws-credentials@v4
//   with:
//     role-to-assume: arn:aws:iam::123456789:role/GitHubDeploy
//     aws-region: us-east-1
```

## Service Control Policies (Organizations)

```json
// SCP — prevent actions across all accounts in OU
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUser",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    },
    {
      "Sid": "RequireIMDSv2",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:MetadataHttpTokens": "required"
        }
      }
    },
    {
      "Sid": "DenyLeavingOrg",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```
