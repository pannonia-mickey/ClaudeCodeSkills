# DevOps Patterns

## Platform Engineering

```
┌────────────────────────────────────────────────────────┐
│                    Developer Portal                     │
│  (Backstage / Port / Cortex)                           │
├────────────────────────────────────────────────────────┤
│    Service Catalog │ Templates │ Docs │ Scorecards     │
├────────────────────────────────────────────────────────┤
│                  Internal Platform                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ CI/CD    │ │ Infra    │ │ Observ.  │ │ Security │ │
│  │ Pipeline │ │ Provisn. │ │ Stack    │ │ Tooling  │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
├────────────────────────────────────────────────────────┤
│              Cloud Infrastructure                       │
│     AWS / Azure / GCP / Kubernetes                     │
└────────────────────────────────────────────────────────┘
```

## Golden Paths

```yaml
# Backstage software template — golden path for new service
# template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: node-microservice
  title: Node.js Microservice
  description: Production-ready Node.js service with CI/CD, monitoring, and docs
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service Details
      properties:
        name:
          type: string
          description: Name of the service (kebab-case)
        owner:
          type: string
          ui:field: OwnerPicker
        system:
          type: string
          ui:field: EntityPicker
  steps:
    - id: scaffold
      name: Scaffold from template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
    - id: publish
      name: Create repository
      action: publish:github
      input:
        repoUrl: github.com?owner=myorg&repo=${{ parameters.name }}
    - id: register
      name: Register in catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
```

## Self-Service Infrastructure

```hcl
# Terraform module for self-service database provisioning
# Developers request via PR, platform team reviews

# modules/managed-database/main.tf
variable "service_name" {
  type = string
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]+$", var.service_name))
    error_message = "Service name must be kebab-case"
  }
}

variable "tier" {
  type    = string
  default = "small"
  validation {
    condition     = contains(["small", "medium", "large"], var.tier)
    error_message = "Tier must be small, medium, or large"
  }
}

locals {
  tiers = {
    small  = { instance_class = "db.t3.micro",  storage = 20 }
    medium = { instance_class = "db.t3.medium", storage = 100 }
    large  = { instance_class = "db.r6g.large", storage = 500 }
  }
}

resource "aws_db_instance" "main" {
  identifier     = "${var.service_name}-db"
  instance_class = local.tiers[var.tier].instance_class
  allocated_storage = local.tiers[var.tier].storage
  # ... security, backup, monitoring config
}
```

## Developer Experience Metrics

```yaml
# DORA Metrics — measure DevOps performance
deployment_frequency:
  elite: "Multiple times per day"
  high: "Daily to weekly"
  medium: "Weekly to monthly"
  low: "Monthly to quarterly"

lead_time_for_changes:
  elite: "< 1 hour"
  high: "1 day to 1 week"
  medium: "1 week to 1 month"
  low: "> 1 month"

change_failure_rate:
  elite: "< 5%"
  high: "5-10%"
  medium: "10-15%"
  low: "> 15%"

time_to_restore:
  elite: "< 1 hour"
  high: "< 1 day"
  medium: "< 1 week"
  low: "> 1 week"

# Track with DORA metrics dashboard
# Tools: LinearB, Sleuth, Faros AI, custom Prometheus metrics
```

## Feature Flags

```typescript
// Feature flag integration for decoupling deploy from release
import { UnleashClient } from 'unleash-proxy-client';

const unleash = new UnleashClient({
  url: 'https://unleash.internal/api/frontend',
  clientKey: process.env.UNLEASH_CLIENT_KEY,
  appName: 'my-service',
});

// Gradual rollout
if (unleash.isEnabled('new-checkout-flow', { userId: user.id })) {
  return newCheckoutFlow(cart);
}
return legacyCheckoutFlow(cart);

// Kill switch for instant rollback without deployment
// Toggle in Unleash UI → immediate effect
```
