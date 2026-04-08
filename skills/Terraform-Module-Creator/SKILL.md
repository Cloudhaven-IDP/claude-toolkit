---
name: terraform-module
description: Create clean, production-ready Terraform modules. Use when creating any Terraform module, scaffolding infrastructure, or working with any provider (AWS, GCP, MongoDB Atlas, CoreWeave, Spacelift, Cloudflare, etc.). Trigger for "create a module", "terraform for X", or any infrastructure-as-code work.
---

# Terraform Module Creator

Keep it simple. Readable code > clever code.

## Core Principles

1. **Readable first** - Anyone should understand it immediately
2. **Secure by default** - Encryption on, protection on
3. **Environment-aware** - Prod gets stricter settings automatically
4. **No magic** - Explicit is better than implicit

## File Structure

```
module-name/
├── main.tf          # Resources
├── variables.tf     # Inputs
├── outputs.tf       # Outputs
├── versions.tf      # Provider versions
└── README.md        # Working example
```

Split files only when it helps readability (iam.tf, data.tf).

## versions.tf

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## variables.tf

Group logically. Every variable needs `description` and `type`:

```hcl
#------------------------------------------------------------------------------
# Required
#------------------------------------------------------------------------------

variable "name" {
  description = "Resource name"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
}

#------------------------------------------------------------------------------
# Optional
#------------------------------------------------------------------------------

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}
```

Add validation when it prevents mistakes:

```hcl
variable "name" {
  description = "Resource name"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]*$", var.name))
    error_message = "Name must be lowercase with hyphens only."
  }
}
```

## config.yaml (Shared Context)

Each leaf directory (`applications/<app>/`, `platform/<component>/`) has a `config.yaml` with shared tags and context. Always read it instead of hardcoding values.

Repo structure:
```
Infrastructure/
├── modules/           # Reusable modules
├── applications/      # App-specific resources (ECR, IAM, secrets)
│   └── <app>/
│       └── config.yaml
├── platform/          # Platform-level resources
│   └── <component>/
│       └── config.yaml
└── README.md
```

Example `config.yaml`:
```yaml
network: "home-network"
managedBy: "terraform"
account: "cloudhaven"
region: "us-east-1"
cluster: "my-cluster"
```

Read it with `yamldecode` and merge into tags:

```hcl
locals {
  config = yamldecode(file("${path.module}/config.yaml"))

  tags = merge({
    Name      = var.name
    ManagedBy = local.config.managedBy
    Account   = local.config.account
    Network   = local.config.network
  }, var.tags)
}
```

This keeps tags consistent without repeating them across every resource.

## main.tf

Use locals for computed values. Keep it clean:

```hcl
locals {
  config  = yamldecode(file("${path.module}/config.yaml"))
  is_prod = var.environment == "prod"

  tags = merge({
    Name        = var.name
    Environment = var.environment
    ManagedBy   = local.config.managedBy
    Account     = local.config.account
  }, var.tags)
}

resource "aws_resource" "this" {
  name = var.name

  # Prod gets protection
  deletion_protection = local.is_prod

  tags = local.tags
}
```

## outputs.tf

Output what consumers need:

```hcl
output "id" {
  description = "Resource ID"
  value       = aws_resource.this.id
}

output "arn" {
  description = "Resource ARN"
  value       = aws_resource.this.arn
}
```

## Patterns

### Wrap Community Modules

Don't reinvent. Wrap with your defaults:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = var.name
  cidr = var.cidr

  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = local.tags
}
```

### Conditional Resources

```hcl
resource "aws_kms_key" "this" {
  count = var.enable_encryption ? 1 : 0

  description = "Encryption key for ${var.name}"
  tags        = local.tags
}
```

### Dynamic Blocks (only when needed)

```hcl
resource "aws_security_group" "this" {
  name   = var.name
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidrs
    }
  }
}
```

### for_each Over count

```hcl
resource "aws_subnet" "private" {
  for_each = var.private_subnets

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = merge(local.tags, { Name = each.key })
}
```

## Example: EventBridge Scheduler

```hcl
# main.tf
locals {
  is_prod = var.environment == "prod"

  tags = merge({
    Name        = var.name
    Environment = var.environment
    ManagedBy   = "terraform"
  }, var.tags)
}

resource "aws_scheduler_schedule" "this" {
  name       = var.name
  group_name = var.group_name

  schedule_expression          = var.schedule_expression
  schedule_expression_timezone = var.timezone

  # Prod is enabled by default, others disabled
  state = var.enabled != null ? (var.enabled ? "ENABLED" : "DISABLED") : (local.is_prod ? "ENABLED" : "DISABLED")

  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = var.target_arn
    role_arn = aws_iam_role.scheduler.arn
  }
}

resource "aws_iam_role" "scheduler" {
  name = "${var.name}-scheduler"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Action    = "sts:AssumeRole"
      Principal = { Service = "scheduler.amazonaws.com" }
    }]
  })

  tags = local.tags
}

resource "aws_iam_role_policy" "scheduler" {
  name = "${var.name}-scheduler"
  role = aws_iam_role.scheduler.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = var.target_actions
      Resource = var.target_arn
    }]
  })
}
```

```hcl
# variables.tf
variable "name" {
  description = "Schedule name"
  type        = string
}

variable "environment" {
  description = "Environment"
  type        = string
}

variable "schedule_expression" {
  description = "Cron or rate expression"
  type        = string
}

variable "target_arn" {
  description = "Target resource ARN"
  type        = string
}

variable "target_actions" {
  description = "IAM actions for target"
  type        = list(string)
  default     = ["lambda:InvokeFunction"]
}

variable "timezone" {
  description = "Schedule timezone"
  type        = string
  default     = "UTC"
}

variable "group_name" {
  description = "Schedule group"
  type        = string
  default     = "default"
}

variable "enabled" {
  description = "Enable schedule (defaults based on environment)"
  type        = bool
  default     = null
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}
```

```hcl
# outputs.tf
output "schedule_arn" {
  description = "Schedule ARN"
  value       = aws_scheduler_schedule.this.arn
}

output "role_arn" {
  description = "Scheduler IAM role ARN"
  value       = aws_iam_role.scheduler.arn
}
```

## Third-Party Providers

Same patterns, different providers:

### MongoDB Atlas

```hcl
terraform {
  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.15"
    }
  }
}

resource "mongodbatlas_cluster" "this" {
  project_id = var.project_id
  name       = var.name

  cluster_type = "REPLICASET"
  provider_name = "AWS"

  # Prod gets bigger instances
  provider_instance_size_name = local.is_prod ? "M30" : "M10"

  # Always enable backups
  cloud_backup = true
}
```

### Spacelift

```hcl
terraform {
  required_providers {
    spacelift = {
      source  = "spacelift-io/spacelift"
      version = "~> 1.0"
    }
  }
}

resource "spacelift_stack" "this" {
  name         = var.name
  repository   = var.repository
  branch       = var.branch
  project_root = var.project_root

  terraform_version = var.terraform_version

  # Prod needs manual approval
  autodeploy = !local.is_prod

  labels = ["env:${var.environment}", "managed-by:terraform"]
}
```

### Any Provider

The pattern is always the same:

1. Pin provider version
2. Use locals for environment logic
3. Secure defaults for prod
4. Clear variable descriptions
5. Useful outputs

## README.md

```markdown
# Module Name

What it creates.

## Usage

\`\`\`hcl
module "example" {
  source = "path/to/module"

  name        = "my-resource"
  environment = "dev"
}
\`\`\`

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| name | Resource name | string | - | yes |
| environment | Environment | string | - | yes |

## Outputs

| Name | Description |
|------|-------------|
| id | Resource ID |
| arn | Resource ARN |
```

## Checklist

Before finishing:

- [ ] Can someone understand this in 30 seconds?
- [ ] Are prod defaults secure?
- [ ] Does the README example actually work?
- [ ] Are all variables described?
- [ ] Are outputs useful to consumers?
