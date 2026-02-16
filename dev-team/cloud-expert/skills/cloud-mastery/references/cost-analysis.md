# Cost Analysis

## AWS Cost Explorer Queries

```
# Top spending services (AWS CLI)
aws ce get-cost-and-usage \
  --time-period Start=2025-01-01,End=2025-02-01 \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# Cost by team (using cost allocation tags)
aws ce get-cost-and-usage \
  --time-period Start=2025-01-01,End=2025-02-01 \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=TAG,Key=team

# Unused resources
aws ce get-cost-and-usage \
  --filter '{
    "Dimensions": {
      "Key": "USAGE_TYPE_GROUP",
      "Values": ["EC2: Running Hours"]
    }
  }' \
  --metrics "UsageQuantity" "BlendedCost"
```

## Tagging Strategy

```hcl
# Required tags for all resources
locals {
  required_tags = {
    Environment = var.environment    # dev, staging, production
    Team        = var.team           # platform, backend, frontend
    Service     = var.service_name   # user-service, payment-api
    CostCenter  = var.cost_center    # engineering, marketing
    ManagedBy   = "terraform"        # terraform, manual, cdk
  }
}

# AWS Tag Policy (Organizations)
# Enforce required tags across all accounts
{
  "tags": {
    "Environment": {
      "tag_key": { "@@assign": "Environment" },
      "tag_value": { "@@assign": ["dev", "staging", "production"] },
      "enforced_for": { "@@assign": ["ec2:instance", "rds:db", "s3:bucket"] }
    }
  }
}
```

## Budget Alerts

```hcl
# Terraform — AWS Budget with alerts
resource "aws_budgets_budget" "monthly" {
  name         = "monthly-budget"
  budget_type  = "COST"
  limit_amount = "10000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_email_addresses = ["platform-team@company.com"]
  }

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = ["platform-lead@company.com"]
  }
}

# Per-service budget using tags
resource "aws_budgets_budget" "per_service" {
  for_each = toset(["user-service", "payment-api", "analytics"])

  name         = "${each.key}-budget"
  budget_type  = "COST"
  limit_amount = "2000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Service$${each.key}"]
  }
}
```

## FinOps Practices

```
Monthly FinOps Review Process:
1. Pull cost report by service and team
2. Identify top 5 cost increases (>10% MoM)
3. Review utilization metrics:
   - EC2: Average CPU < 20% → right-size
   - RDS: Average CPU < 10% → downsize or serverless
   - Lambda: Average duration vs memory → optimize
   - S3: Access patterns → lifecycle policies
4. Check Savings Plans coverage
5. Review spot instance savings opportunities
6. Generate team cost reports for engineering leads

Anomaly Detection:
- AWS Cost Anomaly Detection (free)
- Alert on >20% daily spend increase
- Review within 24 hours
```

## TCO Calculation Template

```
Service: Payment Processing API
Monthly Traffic: 10M requests

Option A: ECS Fargate
  Compute: 2 tasks × 1 vCPU × 2GB × 730hrs = $XXX
  ALB:     $XX + $XX/million LCU-hours
  Total:   ~$XXX/month

Option B: Lambda
  Invocations: 10M × $0.20/1M = $2.00
  Duration:    10M × 200ms × 512MB = $XX
  API Gateway: 10M × $1.00/1M = $10.00
  Total:       ~$XX/month

Option C: EC2 (Reserved)
  Instance: t3.medium RI 1yr = $XXX/month
  ALB:      $XX/month
  Total:    ~$XXX/month

Recommendation: Lambda for < 50M req/month
                ECS Fargate for 50M-500M req/month
                EC2 Reserved for > 500M req/month (steady state)
```

## Cost Optimization Quick Wins

```bash
# Find unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType}' \
  --output table

# Find idle Elastic IPs
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocationId:AllocationId}' \
  --output table

# Find old snapshots (> 90 days)
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[?StartTime<'$(date -d '-90 days' +%Y-%m-%d)'].{ID:SnapshotId,Size:VolumeSize,Date:StartTime}" \
  --output table

# Find underutilized RDS instances
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=my-db \
  --start-time $(date -d '-7 days' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average
```
