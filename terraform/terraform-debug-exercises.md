# Terraform — Environments to debug

Each scenario is an environment that **looks functional** but contains bugs to find.
Some will break `terraform plan`, others `terraform apply`, others will pass but produce an incorrect or insecure result.

---

## Scenario 1 — VPC network stack (AWS)

The team wrote this stack to provision a multi-AZ VPC with public subnets and a bastion.
The code doesn't run. Find and fix all the bugs.

### `variables.tf`

```hcl
variable "project" {
  type    = string
  default = "enalean"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}

variable "public_subnet_cidrs" {
  type = map(string)
  default = {
    "eu-west-1a" = "10.0.1.0/24"
    "eu-west-1b" = "10.0.2.0/24"
    "eu-west-1c" = "10.0.3.0/24"
  }
}

variable "allowed_ssh_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}
```

### `main.tf`

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name    = "${var.project}-vpc"
    Project = var.project
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project}-igw"
  }
}

resource "aws_subnet" "public" {
  for_each = var.availability_zones

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[each.key]
  availability_zone       = each.key
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project}-public-${each.key}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project}-rt-public"
  }
}

resource "aws_route" "internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "bastion" {
  name   = "${var.project}-bastion"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port  = 22
    to_port    = 22
    protocol   = "tcp"
    cidr_block = var.allowed_ssh_cidrs
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Scenario 2 — ECS Fargate application stack + Secrets Manager

The team is deploying a containerized app on ECS Fargate with a database secret injected through Secrets Manager.
`terraform apply` succeeds, but the app doesn't start — and there is an undetected security issue.

### `variables.tf`

```hcl
variable "app_name" {
  type    = string
  default = "tuleap-app"
}

variable "db_password" {
  type      = string
  sensitive = true
}

variable "container_image" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}
```

### `secrets.tf`

```hcl
resource "aws_secretsmanager_secret" "db" {
  name = "/${var.app_name}/db"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = var.db_password
}
```

### `iam.tf`

```hcl
data "aws_iam_policy_document" "ecs_task" {
  statement {
    effect  = "Allow"
    actions = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
    resources = ["*"]
  }

  statement {
    effect  = "Allow"
    actions = ["logs:CreateLogStream", "logs:PutLogEvents"]
    resources = ["${aws_cloudwatch_log_group.ecs.arn}:*"]
  }
}

resource "aws_iam_role" "ecs_task" {
  name = "${var.app_name}-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "ecs_task" {
  role   = aws_iam_role.ecs_task.name
  policy = data.aws_iam_policy_document.ecs_task.json
}
```

### `ecs.tf`

```hcl
resource "aws_cloudwatch_log_group" "app_logs" {
  name              = "/ecs/${var.app_name}"
  retention_in_days = 30
}

resource "aws_ecs_cluster" "main" {
  name = var.app_name
}

resource "aws_ecs_task_definition" "app" {
  family                   = var.app_name
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 256
  memory                   = 512
  execution_role_arn       = aws_iam_role.ecs_task.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = var.app_name
    image = var.container_image
    portMappings = [{
      containerPort = 80
      protocol      = "tcp"
    }]
    secrets = [{
      name      = "DB_PASSWORD"
      valueFrom = aws_secretsmanager_secret_version.db.secret_string
    }]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.app_logs.name
        "awslogs-region"        = "eu-west-1"
        "awslogs-stream-prefix" = "ecs"
      }
    }
  }])
}

resource "aws_ecs_service" "app" {
  name            = var.app_name
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "EC2"

  network_configuration {
    subnets          = var.subnet_ids
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }
}
```

### `outputs.tf`

```hcl
output "cluster_name" {
  value = aws_ecs_cluster.main.name
}

output "db_secret_value" {
  value = aws_secretsmanager_secret_version.db.secret_string
}
```
