# Exercices Terraform

## Exercice 1 — Variables et validation (Facile)

Tu dois configurer un bucket S3 pour stocker des artefacts de build.

**Contexte :** L'équipe veut standardiser les noms de buckets avec le préfixe `enalean-`, une taille de nom entre 10 et 40 caractères, et des environnements restreints à `dev`, `staging`, `prod`.

**Objectif :** Déclare trois variables :
- `bucket_name` (string) : valider que le nom commence par `enalean-`
- `environment` (string) : restreindre aux valeurs `dev`, `staging`, `prod`
- `retention_days` (number) : valeur par défaut à `30`, ne doit pas dépasser `365`

```hcl
# Variables à compléter

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

## Exercice 2 — Locals et expressions conditionnelles (Facile-Moyen)

**Contexte :** Selon l'environnement, les ressources doivent avoir des tags différents et un niveau de logging activé uniquement en prod.

**Objectif :** À partir de la variable `environment` (string), construire dans un bloc `locals` :
- `common_tags` : map avec `Environment`, `ManagedBy = "terraform"`, et `CostCenter` qui vaut `"infra-prod"` si prod, sinon `"infra-dev"`
- `enable_logging` : bool, `true` uniquement si environment est `prod`
- `instance_type` : `"t3.medium"` en dev, `"t3.large"` en staging, `"m5.xlarge"` en prod

```hcl
variable "environment" {
  type = string
}

locals {
  # ...
}
```

---

## Exercice 3 — for_each sur une map (Moyen)

**Contexte :** Tu dois créer plusieurs utilisateurs IAM avec des politiques différentes selon leur rôle.

**Objectif :** À partir de la variable `users` ci-dessous, créer :
1. Une ressource `aws_iam_user` pour chaque utilisateur
2. Une ressource `aws_iam_user_policy_attachment` qui attache la politique correspondante à chaque user
3. Un output `user_arns` qui retourne une map `nom -> ARN`

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

# Compléter avec les ressources et l'output
```

---

## Exercice 4 — Module et dynamic blocks (Moyen-Difficile)

**Contexte :** Tu dois créer un Security Group AWS réutilisable via un module.

**Structure attendue :**
```
modules/security_group/
  main.tf
  variables.tf
  outputs.tf
```

**Objectif :** Le module doit accepter :
- `name` (string) : nom du SG
- `vpc_id` (string) : VPC de destination
- `ingress_rules` : liste d'objets avec `from_port`, `to_port`, `protocol`, `cidr_blocks`
- `egress_rules` : même structure, avec une règle par défaut `0/0` si la liste est vide

Le bloc `ingress` et `egress` de la ressource `aws_security_group` doivent être générés avec des **dynamic blocks**.

L'output doit exposer `security_group_id` et `security_group_arn`.

**Usage attendu du module :**
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

## Exercice 5 — Data sources, moved et refactoring (Difficile)

**Contexte :** Une instance EC2 a été créée manuellement (hors Terraform). Tu dois l'importer dans l'état, puis refactoriser le code pour la déplacer dans un module sans recréer la ressource.

**Partie A — Import**

L'instance a l'ID `i-0abc123def456`. Écris la commande d'import ET le bloc `import` HCL (Terraform ≥ 1.5) pour la ressource suivante :

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

**Partie B — Refactoring avec `moved`**

Après l'import, tu veux déplacer `aws_instance.web` dans un module `module.web_server`. Écris le bloc `moved` qui permet ce déplacement sans détruire/recréer la ressource.

**Partie C — Question ouverte**

Quelle est la différence entre un `moved` block et un `terraform state mv` ? Dans quel cas préfères-tu l'un ou l'autre ?
