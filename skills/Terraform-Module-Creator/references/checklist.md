# Module Checklist

Quick checklist before finishing a module.

## Files

- [ ] main.tf
- [ ] variables.tf
- [ ] outputs.tf
- [ ] versions.tf
- [ ] README.md with working example

## Variables

- [ ] Every variable has `description`
- [ ] Every variable has `type`
- [ ] Sensible defaults where appropriate
- [ ] Validation for formats that matter

## Security

- [ ] Encryption enabled by default
- [ ] Prod gets deletion protection
- [ ] No hardcoded secrets
- [ ] IAM follows least privilege

## Quality

- [ ] Someone can understand it in 30 seconds
- [ ] No unnecessary complexity
- [ ] Locals used for computed values
- [ ] Tags on all resources
- [ ] Outputs are useful

## Documentation

- [ ] README describes what it creates
- [ ] Usage example actually works
- [ ] Inputs table
- [ ] Outputs table

## Template

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# variables.tf
variable "name" {
  description = "Resource name"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}

# main.tf
locals {
  is_prod = var.environment == "prod"

  tags = merge({
    Name        = var.name
    Environment = var.environment
    ManagedBy   = "terraform"
  }, var.tags)
}

resource "aws_resource" "this" {
  name                = var.name
  deletion_protection = local.is_prod
  tags                = local.tags
}

# outputs.tf
output "id" {
  description = "Resource ID"
  value       = aws_resource.this.id
}

output "arn" {
  description = "Resource ARN"
  value       = aws_resource.this.arn
}
```
