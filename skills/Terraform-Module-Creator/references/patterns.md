# Terraform Patterns Reference

Quick reference for common patterns. Keep it simple.

## Locals

Use for computed values and to avoid repetition:

```hcl
locals {
  is_prod = var.environment == "prod"

  tags = merge({
    Name        = var.name
    Environment = var.environment
    ManagedBy   = "terraform"
  }, var.tags)
}
```

## Conditional Resources

On/off with count:

```hcl
resource "aws_kms_key" "this" {
  count = var.enable_encryption ? 1 : 0
  # ...
}

# Reference safely
output "kms_key_arn" {
  value = var.enable_encryption ? aws_kms_key.this[0].arn : null
}
```

## Multiple Resources

Use for_each with maps:

```hcl
variable "buckets" {
  type = map(object({
    versioning = bool
  }))
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = each.key
}

resource "aws_s3_bucket_versioning" "this" {
  for_each = { for k, v in var.buckets : k => v if v.versioning }
  bucket   = aws_s3_bucket.this[each.key].id

  versioning_configuration {
    status = "Enabled"
  }
}
```

## Dynamic Blocks

When you have repeating nested blocks:

```hcl
variable "ingress_rules" {
  type = list(object({
    port  = number
    cidrs = list(string)
  }))
}

resource "aws_security_group" "this" {
  name = var.name

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

## Smart Defaults

Let environment drive defaults:

```hcl
variable "backup_retention" {
  description = "Backup retention days (default: 30 for prod, 7 otherwise)"
  type        = number
  default     = null
}

locals {
  backup_retention = coalesce(var.backup_retention, local.is_prod ? 30 : 7)
}
```

## Validation

Catch errors early:

```hcl
variable "name" {
  type = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]*$", var.name))
    error_message = "Name must be lowercase with hyphens."
  }
}

variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}
```

## Lifecycle

Protect important resources:

```hcl
resource "aws_db_instance" "this" {
  # ...

  lifecycle {
    prevent_destroy = true        # Can't accidentally delete
    ignore_changes  = [password]  # Managed elsewhere
  }
}
```

## Moved Blocks

Rename without destroying:

```hcl
moved {
  from = aws_s3_bucket.old_name
  to   = aws_s3_bucket.this
}
```

## Data Sources

Look up existing resources:

```hcl
data "aws_vpc" "this" {
  filter {
    name   = "tag:Name"
    values = [var.vpc_name]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.this.id]
  }

  tags = {
    Tier = "private"
  }
}
```

## IAM Policies

Keep them simple and scoped:

```hcl
data "aws_iam_policy_document" "read" {
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:ListBucket"]
    resources = [aws_s3_bucket.this.arn, "${aws_s3_bucket.this.arn}/*"]
  }
}

resource "aws_iam_policy" "read" {
  name   = "${var.name}-read"
  policy = data.aws_iam_policy_document.read.json
}
```

## Outputs

Be generous with outputs:

```hcl
output "id" {
  description = "Resource ID"
  value       = aws_resource.this.id
}

output "arn" {
  description = "Resource ARN"
  value       = aws_resource.this.arn
}

# Conditional
output "endpoint" {
  description = "Endpoint (if created)"
  value       = var.create_endpoint ? aws_endpoint.this[0].dns : null
}

# Sensitive
output "password" {
  description = "Generated password"
  value       = random_password.this.result
  sensitive   = true
}
```

## Provider Examples

### MongoDB Atlas

```hcl
resource "mongodbatlas_cluster" "this" {
  project_id                  = var.project_id
  name                        = var.name
  cluster_type                = "REPLICASET"
  provider_name               = "AWS"
  provider_instance_size_name = local.is_prod ? "M30" : "M10"
  cloud_backup                = true
}
```

### Spacelift

```hcl
resource "spacelift_stack" "this" {
  name         = var.name
  repository   = var.repository
  branch       = var.branch
  autodeploy   = !local.is_prod
  labels       = ["env:${var.environment}"]
}
```

### Cloudflare

```hcl
resource "cloudflare_record" "this" {
  zone_id = var.zone_id
  name    = var.subdomain
  type    = "CNAME"
  value   = var.target
  proxied = true
}
```

### Datadog

```hcl
resource "datadog_monitor" "this" {
  name    = "[${upper(var.environment)}] ${var.name}"
  type    = "metric alert"
  query   = var.query
  message = var.message

  tags = ["env:${var.environment}"]
}
```

## Anti-Patterns

**Don't do this:**

```hcl
# Too clever - hard to understand
locals {
  x = flatten([for a in var.a : [for b in a.b : [for c in b.c : { ... }]]])
}

# Magic numbers
resource "aws_instance" "this" {
  count = 3  # Why 3?
}

# Hardcoded values
resource "aws_s3_bucket" "this" {
  bucket = "my-company-prod-data"  # Use variables
}
```

**Do this instead:**

```hcl
# Clear and simple
locals {
  subnets = { for s in var.subnets : s.name => s }
}

# Self-documenting
variable "instance_count" {
  description = "Number of instances"
  default     = 3
}

# Configurable
variable "bucket_name" {
  description = "S3 bucket name"
  type        = string
}
```
