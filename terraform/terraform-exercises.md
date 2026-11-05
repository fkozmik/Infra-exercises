# Terraform Exercises

## Exercise 1 — Variables and validation (Easy)

You need to configure an S3 bucket to store build artifacts.

**Context:** The team wants to standardize bucket names with the `enalean-` prefix, a name length between 10 and 40 characters, and environments restricted to `dev`, `staging`, `prod`.

**Goal:** Declare three variables:
- `bucket_name` (string): validate that the name starts with `enalean-`
- `environment` (string): restrict to the values `dev`, `staging`, `prod`
- `retention_days` (number): default to `30`, must not exceed `365`

```hcl
# Variables to complete

variable "bucket_name" {
  # ...
}

variable "environment" {
  # ...
}

variable "retention_days" {
  # ...
}
```

---

## Exercise 2 — Locals and conditional expressions (Easy-Medium)

**Context:** Depending on the environment, resources must carry different tags, and logging must only be enabled in prod.

**Goal:** From the `environment` variable (string), build a `locals` block containing:
- `common_tags`: a map with `Environment`, `ManagedBy = "terraform"`, and `CostCenter` set to `"infra-prod"` in prod, `"infra-dev"` otherwise
- `enable_logging`: bool, `true` only if environment is `prod`
- `instance_type`: `"t3.medium"` in dev, `"t3.large"` in staging, `"m5.xlarge"` in prod

```hcl
variable "environment" {
  type = string
}

locals {
  # ...
}
```

---

## Exercise 3 — for_each over a map (Medium)

**Context:** You need to create several IAM users with different policies depending on their role.

**Goal:** From the `users` variable below, create:
1. An `aws_iam_user` resource for each user
2. An `aws_iam_user_policy_attachment` resource attaching the matching policy to each user
3. A `user_arns` output returning a `name -> ARN` map

```hcl
variable "users" {
  type = map(object({
    policy_arn = string
    force_destroy = bool
  }))
  default = {
    alice = {
      policy_arn    = "arn:aws:iam::aws:policy/ReadOnlyAccess"
      force_destroy = false
    }
    bob = {
      policy_arn    = "arn:aws:iam::aws:policy/PowerUserAccess"
      force_destroy = true
    }
  }
}

# Complete with the resources and the output
```

---

## Exercise 4 — Module and dynamic blocks (Medium-Hard)

**Context:** You need to build a reusable AWS Security Group as a module.

**Expected structure:**
```
modules/security_group/
  main.tf
  variables.tf
  outputs.tf
```

**Goal:** The module must accept:
- `name` (string): SG name
- `vpc_id` (string): target VPC
- `ingress_rules`: list of objects with `from_port`, `to_port`, `protocol`, `cidr_blocks`
- `egress_rules`: same structure, with a default `0/0` rule if the list is empty

The `ingress` and `egress` blocks of the `aws_security_group` resource must be generated with **dynamic blocks**.

The outputs must expose `security_group_id` and `security_group_arn`.

**Expected module usage:**
```hcl
module "web_sg" {
  source = "./modules/security_group"

  name   = "web-server"
  vpc_id = "vpc-12345"

  ingress_rules = [
    { from_port = 443, to_port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { from_port = 80,  to_port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
  ]

  egress_rules = []
}
```

---

## Exercise 5 — Data sources, moved and refactoring (Hard)

**Context:** An EC2 instance was created manually (outside Terraform). You need to import it into state, then refactor the code to move it into a module without recreating the resource.

**Part A — Import**

The instance has ID `i-0abc123def456`. Write the import command AND the HCL `import` block (Terraform ≥ 1.5) for the following resource:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

**Part B — Refactoring with `moved`**

After the import, you want to move `aws_instance.web` into a `module.web_server` module. Write the `moved` block that performs this move without destroying/recreating the resource.

**Part C — Open question**

What is the difference between a `moved` block and `terraform state mv`? When would you prefer one over the other?
