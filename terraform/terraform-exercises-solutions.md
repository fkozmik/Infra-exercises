# Solutions — Exercices Terraform

## Exercice 1 — Variables et validation

```hcl
variable "bucket_name" {
  type        = string
  description = "Nom du bucket S3"

  validation {
    condition     = startswith(var.bucket_name, "enalean-")
    error_message = "Le nom du bucket doit commencer par 'enalean-'."
  }

  validation {
    condition     = length(var.bucket_name) >= 10 && length(var.bucket_name) <= 40
    error_message = "Le nom doit faire entre 10 et 40 caractères."
  }
}

variable "environment" {
  type        = string
  description = "Environnement cible"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "L'environnement doit être 'dev', 'staging' ou 'prod'."
  }
}

variable "retention_days" {
  type        = number
  description = "Durée de rétention des objets en jours"
  default     = 30

  validation {
    condition     = var.retention_days >= 1 && var.retention_days <= 365
    error_message = "La rétention doit être entre 1 et 365 jours."
  }
}
```

**Points clés :**
- Plusieurs blocs `validation` sont possibles sur une même variable — préférer des messages d'erreur distincts par règle plutôt qu'une condition composée illisible.
- `startswith()` est disponible depuis Terraform 1.3. Avant : `substr(var.bucket_name, 0, 8) == "enalean-"`.

---

## Exercice 2 — Locals et expressions conditionnelles

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

**Points clés :**
- La lookup via `map[key]` est plus lisible qu'une chaîne de ternaires imbriqués. Elle plantera au plan si `environment` n'est pas dans la map — ce qui est souhaitable si la variable n'est pas validée en amont.
- Alternative avec `lookup()` si une valeur par défaut est nécessaire : `lookup(local._types, var.environment, "t3.medium")`.

---

## Exercice 3 — for_each sur une map

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

**Points clés :**
- `for_each` sur une map utilise `each.key` / `each.value` — l'adresse de la ressource dans le state sera `aws_iam_user.this["alice"]` plutôt qu'un index numérique. Ajouter un user ne recréera pas les autres (contrairement à `count`).
- La dépendance entre `aws_iam_user_policy_attachment` et `aws_iam_user` est implicite via `aws_iam_user.this[each.key].name` — pas besoin de `depends_on`.

---

## Exercice 4 — Module et dynamic blocks

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

**Points clés :**
- La logique de fallback egress est dans `locals`, pas dans la ressource — plus testable et lisible.
- L'itérateur par défaut d'un `dynamic` block porte le nom du block (`ingress`, `egress`). On peut le renommer avec `iterator = rule` si plusieurs dynamic blocks imbriqués risquent de collisionner.

---

## Exercice 5 — Import, moved et refactoring

### Partie A — Import

**Commande CLI (Terraform < 1.5 ou fallback) :**
```bash
terraform import aws_instance.web i-0abc123def456
```

**Bloc HCL (Terraform ≥ 1.5) :**
```hcl
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}
```

L'avantage du bloc HCL : l'import est versionné dans le code, visible en `plan`, et peut être couplé à un `terraform plan -generate-config-out=generated.tf` pour générer la configuration de la ressource automatiquement.

---

### Partie B — Moved block

```hcl
moved {
  from = aws_instance.web
  to   = module.web_server.aws_instance.this
}
```

Après application, supprimer le bloc `moved` (il n'est utile que le temps d'un plan/apply).

---

### Partie C — `moved` block vs `terraform state mv`

| | `moved` block | `terraform state mv` |
|---|---|---|
| Versionné | Oui, dans le code | Non, opération manuelle |
| Visible en `plan` | Oui | Non |
| Collaboratif | Oui — tous ceux qui font `apply` l'appliquent | Non — state modifié localement ou via CI |
| Rollback | Supprimer le bloc et ré-appliquer | `terraform state mv` inverse |
| Disponibilité | Terraform ≥ 1.1 | Toutes versions |

**Quand préférer `terraform state mv` :**
- Migration urgente sans passer par un cycle plan/apply complet (ex: incident).
- Terraform < 1.1.
- Besoin de déplacer des ressources entre **workspaces ou states différents** — `moved` ne supporte pas le cross-state.
