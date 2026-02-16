# Terraform Patterns

## Module Composition

```hcl
# environments/production/main.tf — composing modules
module "networking" {
  source      = "../../modules/networking"
  environment = "production"
  vpc_cidr    = "10.0.0.0/16"
  azs         = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

module "database" {
  source           = "../../modules/database"
  environment      = "production"
  vpc_id           = module.networking.vpc_id
  subnet_ids       = module.networking.private_subnet_ids
  instance_class   = "db.r6g.large"
  multi_az         = true
  backup_retention = 30
}

module "compute" {
  source     = "../../modules/compute"
  environment = "production"
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
  db_host    = module.database.endpoint
  db_port    = module.database.port
}
```

## Dynamic Blocks

```hcl
# Dynamic security group rules
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    { port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"], description = "HTTP" },
    { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"], description = "HTTPS" },
  ]
}

resource "aws_security_group" "web" {
  name   = "${var.environment}-web-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## for_each vs count

```hcl
# Prefer for_each — stable keys, no index shifting on changes
# Use for_each for resources that are identified by a key
variable "services" {
  type = map(object({
    port     = number
    replicas = number
  }))
  default = {
    api     = { port = 3000, replicas = 3 }
    worker  = { port = 0,    replicas = 2 }
    gateway = { port = 8080, replicas = 2 }
  }
}

resource "aws_ecs_service" "services" {
  for_each     = var.services
  name         = each.key
  desired_count = each.value.replicas
  # Removing "worker" from map won't affect "api" or "gateway" resources
}

# Use count only for conditional creation
resource "aws_cloudwatch_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0
  # ...
}
```

## Import Existing Resources

```hcl
# Terraform 1.5+ import blocks (declarative)
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}

resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
  # Run: terraform plan -generate-config-out=generated.tf
  # This generates the resource configuration from the imported state
}

# Moved blocks — refactoring without recreation
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

## Terragrunt DRY Configuration

```hcl
# terragrunt.hcl — root
remote_state {
  backend = "s3"
  generate = { path = "backend.tf", if_exists = "overwrite" }
  config = {
    bucket         = "mycompany-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# environments/production/app/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/app"
}

inputs = {
  environment    = "production"
  instance_count = 3
  instance_type  = "t3.medium"
}
```

## Local Values and Data Sources

```hcl
# Common tagging pattern
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
    Team        = var.team
    CostCenter  = var.cost_center
  }
}

# Data sources — reference existing resources
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
```
