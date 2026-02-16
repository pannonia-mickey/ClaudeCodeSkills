---
name: Infrastructure as Code
description: This skill should be used when the user asks about "Terraform", "Pulumi", "CloudFormation", "Infrastructure as Code", "IaC", "Terraform modules", "state management", "drift detection", "infrastructure provisioning", or "Terragrunt". It covers Terraform, Pulumi, CloudFormation patterns, state management, and modular IaC design.
---

# Infrastructure as Code

## Terraform Module Structure

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
├── modules/
│   ├── networking/    # VPC, subnets, security groups
│   ├── compute/       # ECS, EC2, Lambda
│   ├── database/      # RDS, DynamoDB
│   └── monitoring/    # CloudWatch, alarms
└── shared/
    ├── backend.tf     # S3 + DynamoDB state config
    └── providers.tf   # Provider versions
```

## Terraform Best Practices

```hcl
# modules/networking/main.tf
terraform {
  required_version = ">= 1.7"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

output "vpc_id" {
  value       = aws_vpc.main.id
  description = "The ID of the VPC"
}
```

## State Management

```hcl
# S3 backend with state locking
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "environments/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# State locking table
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Pulumi (TypeScript)

```typescript
import * as pulumi from '@pulumi/pulumi';
import * as aws from '@pulumi/aws';

const config = new pulumi.Config();
const environment = pulumi.getStack(); // dev, staging, prod

const vpc = new aws.ec2.Vpc('main', {
  cidrBlock: '10.0.0.0/16',
  enableDnsHostnames: true,
  tags: { Name: `${environment}-vpc`, Environment: environment },
});

const subnet = new aws.ec2.Subnet('app', {
  vpcId: vpc.id,
  cidrBlock: '10.0.1.0/24',
  availabilityZone: 'us-east-1a',
  tags: { Name: `${environment}-app-subnet` },
});

export const vpcId = vpc.id;
export const subnetId = subnet.id;
```

## Drift Detection in CI

```yaml
# Terraform plan on PR, apply on merge
terraform-plan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: hashicorp/setup-terraform@v3
    - run: terraform init
    - run: terraform plan -out=tfplan -no-color
      id: plan
    - uses: actions/github-script@v7
      if: github.event_name == 'pull_request'
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `## Terraform Plan\n\`\`\`\n${{ steps.plan.outputs.stdout }}\n\`\`\``
          })
```

## References

- [Terraform Patterns](references/terraform-patterns.md) — Module composition, data sources, dynamic blocks, for_each vs count, import, moved blocks.
- [IaC Testing](references/iac-testing.md) — Terratest, Checkov, tflint, infracost, policy-as-code with OPA/Sentinel.
