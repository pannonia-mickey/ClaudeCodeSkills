# IaC Testing

## Terratest (Go)

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/networking",
        Vars: map[string]interface{}{
            "environment": "test",
            "vpc_cidr":    "10.99.0.0/16",
        },
    }

    // Clean up after test
    defer terraform.Destroy(t, terraformOptions)

    // Deploy
    terraform.InitAndApply(t, terraformOptions)

    // Validate outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)

    // Validate AWS resources
    vpc := aws.GetVpcById(t, vpcId, "us-east-1")
    assert.Equal(t, "10.99.0.0/16", vpc.CidrBlock)

    subnets := aws.GetSubnetsForVpc(t, vpcId, "us-east-1")
    assert.Equal(t, 3, len(subnets))
}
```

## Checkov — Static Analysis

```yaml
# CI integration
- name: Checkov IaC scan
  uses: bridgecrewio/checkov-action@master
  with:
    directory: infrastructure/
    framework: terraform
    quiet: true
    soft_fail: false
    output_format: sarif

# Common checks:
# CKV_AWS_79:  Ensure Instance Metadata Service Version 1 is not enabled
# CKV_AWS_145: Ensure S3 bucket is encrypted with KMS
# CKV_AWS_18:  Ensure S3 bucket has access logging enabled
# CKV_AWS_23:  Ensure every security group has a description
# CKV2_AWS_6:  Ensure S3 bucket has public access block

# Custom policy (.checkov/custom_policy.yaml)
# - Check for required tags
# - Enforce encryption
# - Block public access
```

```yaml
# .checkov.yml — skip specific checks with justification
skip-check:
  - id: CKV_AWS_18
    comment: "Access logging not needed for static asset bucket"
```

## tflint — Terraform Linter

```hcl
# .tflint.hcl
plugin "aws" {
  enabled = true
  version = "0.31.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "aws_instance_invalid_type" {
  enabled = true
}

# Run: tflint --init && tflint
```

## Infracost — Cost Estimation

```yaml
# Show infrastructure cost changes on PRs
- name: Infracost
  uses: infracost/actions/setup@v3
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}

- name: Generate cost diff
  run: |
    infracost diff \
      --path=infrastructure/ \
      --format=json \
      --out-file=/tmp/infracost.json

- name: Post cost comment
  uses: infracost/actions/comment@v1
  with:
    path: /tmp/infracost.json
    behavior: update
    # Shows: "Monthly cost will increase by $45 (+12%)"
```

## Policy-as-Code with OPA

```rego
# policy/terraform.rego
package terraform

# Deny public S3 buckets
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket"
    resource.change.after.acl == "public-read"
    msg := sprintf("S3 bucket '%s' must not be public", [resource.name])
}

# Require encryption on RDS
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    not resource.change.after.storage_encrypted
    msg := sprintf("RDS instance '%s' must have encryption enabled", [resource.name])
}

# Enforce tagging
deny[msg] {
    resource := input.resource_changes[_]
    required_tags := {"Environment", "Team", "ManagedBy"}
    tags := {tag | resource.change.after.tags[tag]}
    missing := required_tags - tags
    count(missing) > 0
    msg := sprintf("Resource '%s' missing required tags: %v", [resource.name, missing])
}
```

```yaml
# CI integration
- name: Terraform plan JSON
  run: terraform plan -out=tfplan && terraform show -json tfplan > plan.json

- name: OPA policy check
  run: |
    opa eval --data policy/ --input plan.json \
      'data.terraform.deny' --format pretty
    # Fails if any deny rules match
```

## Terraform Test Framework (1.6+)

```hcl
# tests/vpc.tftest.hcl — native Terraform testing
run "create_vpc" {
  command = apply

  variables {
    environment = "test"
    vpc_cidr    = "10.99.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.99.0.0/16"
    error_message = "VPC CIDR block did not match expected value"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}

run "verify_subnets" {
  command = plan

  assert {
    condition     = length(aws_subnet.private) == 3
    error_message = "Expected 3 private subnets"
  }
}
```
