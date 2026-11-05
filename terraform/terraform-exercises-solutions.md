# Solutions — Terraform Exercises

## Exercise 1 — Variables and validation

```hcl
variable "bucket_name" {
  type        = string
  description = "S3 bucket name"

  validation {
    condition     = startswith(var.bucket_name, "enalean-")
    error_message = "The bucket name must start with 'enalean-'."
  }

  validation {
    condition     = length(var.bucket_name) >= 10 && length(var.bucket_name) <= 40
    error_message = "The name must be between 10 and 40 characters long."
  }
}

variable "environment" {
  type        = string
  description = "Target environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "The environment must be 'dev', 'staging' or 'prod'."
  }
}

variable "retention_days" {
  type        = number
  description = "Object retention period in days"
  default     = 30

  validation {
    condition     = var.retention_days >= 1 && var.retention_days <= 365
    error_message = "Retention must be between 1 and 365 days."
  }
}
```

**Key points:**
- Several `validation` blocks are allowed on the same variable — prefer distinct error messages per rule over one unreadable compound condition.
- `startswith()` is available since Terraform 1.3. Before that: `substr(var.bucket_name, 0, 8) == "enalean-"`.

---

## Exercise 2 — Locals and conditional expressions

```hcl
variable "environment" {
  type = string
}

locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    CostCenter  = var.environment == "prod" ? "infra-prod" : "infra-dev"
  }

  enable_logging = var.environment == "prod"

  instance_type = {
    dev     = "t3.medium"
    staging = "t3.large"
    prod    = "m5.xlarge"
  }[var.environment]
}
```

**Key points:**
- A `map[key]` lookup is more readable than a chain of nested ternaries. It will fail at plan time if `environment` is not in the map — which is desirable when the variable isn't validated upstream.
- Alternative with `lookup()` when a default value is needed: `lookup(local._types, var.environment, "t3.medium")`.

---

## Exercise 3 — for_each over a map

```hcl
variable "users" {
  type = map(object({
    policy_arn    = string
    force_destroy = bool
  }))
}

resource "aws_iam_user" "this" {
  for_each = var.users

  name          = each.key
  force_destroy = each.value.force_destroy
}

resource "aws_iam_user_policy_attachment" "this" {
  for_each = var.users

  user       = aws_iam_user.this[each.key].name
  policy_arn = each.value.policy_arn
}

output "user_arns" {
  value = { for name, user in aws_iam_user.this : name => user.arn }
}
```

**Key points:**
- `for_each` over a map uses `each.key` / `each.value` — the resource address in state will be `aws_iam_user.this["alice"]` rather than a numeric index. Adding a user won't recreate the others (unlike `count`).
- The dependency between `aws_iam_user_policy_attachment` and `aws_iam_user` is implicit through `aws_iam_user.this[each.key].name` — no `depends_on` needed.

---

## Exercise 4 — Module and dynamic blocks

**`modules/security_group/variables.tf`**
```hcl
variable "name" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}

variable "egress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}
```

**`modules/security_group/main.tf`**
```hcl
locals {
  egress_rules = length(var.egress_rules) > 0 ? var.egress_rules : [
    {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
}

resource "aws_security_group" "this" {
  name   = var.name
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  dynamic "egress" {
    for_each = local.egress_rules
    content {
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }
}
```

**`modules/security_group/outputs.tf`**
```hcl
output "security_group_id" {
  value = aws_security_group.this.id
}

output "security_group_arn" {
  value = aws_security_group.this.arn
}
```

**Key points:**
- The egress fallback logic lives in `locals`, not in the resource — more testable and more readable.
- A `dynamic` block's default iterator is named after the block (`ingress`, `egress`). It can be renamed with `iterator = rule` when nested dynamic blocks risk colliding.

---

## Exercise 5 — Import, moved and refactoring

### Part A — Import

**CLI command (Terraform < 1.5, or as a fallback):**
```bash
terraform import aws_instance.web i-0abc123def456
```

**HCL block (Terraform ≥ 1.5):**
```hcl
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}
```

The advantage of the HCL block: the import is versioned in code, visible in `plan`, and can be paired with `terraform plan -generate-config-out=generated.tf` to generate the resource configuration automatically.

---

### Part B — Moved block

```hcl
moved {
  from = aws_instance.web
  to   = module.web_server.aws_instance.this
}
```

Once applied, remove the `moved` block (it's only useful for the duration of one plan/apply).

---

### Part C — `moved` block vs `terraform state mv`

| | `moved` block | `terraform state mv` |
|---|---|---|
| Versioned | Yes, in code | No, manual operation |
| Visible in `plan` | Yes | No |
| Collaborative | Yes — everyone who runs `apply` applies it | No — state modified locally or via CI |
| Rollback | Remove the block and re-apply | Reverse `terraform state mv` |
| Availability | Terraform ≥ 1.1 | All versions |

**When to prefer `terraform state mv`:**
- Urgent migration without going through a full plan/apply cycle (e.g. during an incident).
- Terraform < 1.1.
- Moving resources between **different workspaces or states** — `moved` doesn't support cross-state moves.
