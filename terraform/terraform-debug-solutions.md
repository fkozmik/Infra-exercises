# Solutions — Environments to debug

---

## Scenario 1 — VPC network stack

4 bugs in total. All of them block `terraform plan`.

---

### Bug 1 — `for_each` over a `list(string)`

**File:** `main.tf` — `aws_subnet.public`

```hcl
# Broken
resource "aws_subnet" "public" {
  for_each = var.availability_zones   # list(string) → error
  ...
}

# Fixed — option A: toset() at the call site
resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)
  ...
}

# Fixed — option B: change the variable type (preferable)
variable "availability_zones" {
  type    = set(string)
  default = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}
```

`for_each` requires a `set` or a `map`. A `list` is rejected because ordering is not a stable identity key in state. `toset()` also deduplicates implicitly.

---

### Bug 2 — Wrong reference to the Internet Gateway

**File:** `main.tf` — `aws_route.internet`

```hcl
# Broken
gateway_id = aws_internet_gateway.main.id   # "main" doesn't exist

# Fixed
gateway_id = aws_internet_gateway.gw.id     # the resource is named "gw"
```

Caught at plan time: `Reference to undeclared resource`. Watch for this especially when renaming resources — state doesn't follow automatically.

---

### Bug 3 — Scalar reference in a `for_each` context

**File:** `main.tf` — `aws_route_table_association.public`

```hcl
# Broken
resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id = aws_subnet.public.id   # scalar reference → error in a for_each context
  ...
}

# Fixed
resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id = each.value.id   # each.value is the current aws_subnet object
  ...
}
```

When a resource is created with `for_each`, it no longer has direct attributes — `aws_subnet.public.id` is invalid. You must go through `each.value` inside the iteration, or `aws_subnet.public["key"].id` from outside.

---

### Bug 4 — `cidr_block` attribute instead of `cidr_blocks`

**File:** `main.tf` — `aws_security_group.bastion`

```hcl
# Broken
ingress {
  from_port  = 22
  to_port    = 22
  protocol   = "tcp"
  cidr_block = var.allowed_ssh_cidrs   # no such attribute on an inline ingress
}

# Fixed
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = var.allowed_ssh_cidrs   # plural, accepts list(string)
}
```

`cidr_block` (singular) is the attribute on `aws_vpc` and `aws_subnet`. On ingress/egress rules it's `cidr_blocks` (plural, `list(string)`). A classic mistake caused by the naming inconsistency in the AWS provider.

---

## Scenario 2 — ECS Fargate + Secrets Manager stack

5 bugs: 3 blocking or silently incorrect, 1 that breaks the app at runtime, 1 security issue.

---

### Bug 1 — Wrong log group reference in the IAM policy

**File:** `iam.tf` — `data.aws_iam_policy_document.ecs_task`

```hcl
# Broken
resources = ["${aws_cloudwatch_log_group.ecs.arn}:*"]
#                                          ^^^
# The resource is named "app_logs", not "ecs"

# Fixed
resources = ["${aws_cloudwatch_log_group.app_logs.arn}:*"]
```

Caught at plan time: `Reference to undeclared resource`. Same root cause as bug 2 in scenario 1 — a rename that wasn't propagated.

---

### Bug 2 — Overly permissive IAM (security bug)

**File:** `iam.tf` — `data.aws_iam_policy_document.ecs_task`

```hcl
# Broken — access to every secret in the account
resources = ["*"]

# Fixed — scoped to the app's secret
resources = [aws_secretsmanager_secret.db.arn]
```

`apply` succeeds and the app works, but the task role can read **every secret in the AWS account**. If the container is compromised, lateral escalation is trivial. Least privilege: the exact secret ARN is enough.

---

### Bug 3 — `valueFrom` points at the secret value, not its ARN

**File:** `ecs.tf` — `aws_ecs_task_definition.app`, `secrets` block

```hcl
# Broken — injects the raw secret value (a Terraform string) as an ARN
secrets = [{
  name      = "DB_PASSWORD"
  valueFrom = aws_secretsmanager_secret_version.db.secret_string
}]

# Fixed — valueFrom expects a Secrets Manager or SSM Parameter Store ARN
secrets = [{
  name      = "DB_PASSWORD"
  valueFrom = aws_secretsmanager_secret.db.arn
}]
```

`valueFrom` in ECS is an ARN, not a value. ECS itself calls Secrets Manager at container startup to fetch the secret. With the raw value, ECS will try to treat it as an invalid ARN → the container never starts. `apply` succeeds because Terraform doesn't validate the contents of the `container_definitions` JSON.

---

### Bug 4 — `launch_type = "EC2"` on a Fargate service

**File:** `ecs.tf` — `aws_ecs_service.app`

```hcl
# Broken
launch_type = "EC2"

# Fixed
launch_type = "FARGATE"
```

The task definition has `requires_compatibilities = ["FARGATE"]` and `network_mode = "awsvpc"`. With `launch_type = "EC2"`, AWS will reject the deployment at apply time (not at plan): the cluster has no EC2 instances, and `awsvpc` without Fargate needs extra configuration. The error only shows up after `apply`.

---

### Bug 5 — Outputting a sensitive value without `sensitive = true`

**File:** `outputs.tf`

```hcl
# Broken — prints the password in cleartext in the terminal and CI logs
output "db_secret_value" {
  value = aws_secretsmanager_secret_version.db.secret_string
}

# Fixed
output "db_secret_value" {
  value     = aws_secretsmanager_secret_version.db.secret_string
  sensitive = true
}
```

Without `sensitive = true`, Terraform prints the value in cleartext in `terraform output` and in CI pipeline logs. The `db_password` variable is marked `sensitive`, but that doesn't propagate automatically to the output. Since Terraform 0.15 a warning is emitted, but not an error — it goes through.

> **Bonus:** the output is named `db_secret_value` but exposes `secret_string` (the cleartext password) rather than `secret_arn`. The ARN is probably what the caller should receive so it can call Secrets Manager itself.
