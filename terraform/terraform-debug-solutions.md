# Solutions — Environnements à debugger

---

## Scénario 1 — Stack réseau VPC

4 bugs au total. Tous bloquent `terraform plan`.

---

### Bug 1 — `for_each` sur une `list(string)`

**Fichier :** `main.tf` — `aws_subnet.public`

```hcl
# Cassé
resource "aws_subnet" "public" {
  for_each = var.availability_zones   # list(string) → erreur
  ...
}

# Corrigé — option A : toset() à l'usage
resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)
  ...
}

# Corrigé — option B : changer le type de la variable (préférable)
variable "availability_zones" {
  type    = set(string)
  default = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}
```

`for_each` exige un `set` ou une `map`. Une `list` est refusée car l'ordre n'est pas une clé d'identité stable dans le state. `toset()` déduplique aussi implicitement.

---

### Bug 2 — Mauvaise référence à l'Internet Gateway

**Fichier :** `main.tf` — `aws_route.internet`

```hcl
# Cassé
gateway_id = aws_internet_gateway.main.id   # "main" n'existe pas

# Corrigé
gateway_id = aws_internet_gateway.gw.id     # la ressource s'appelle "gw"
```

Erreur détectée au plan : `Reference to undeclared resource`. À surveiller particulièrement lors de renommages de ressources — le state ne suit pas automatiquement.

---

### Bug 3 — Référence scalaire dans un contexte `for_each`

**Fichier :** `main.tf` — `aws_route_table_association.public`

```hcl
# Cassé
resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id = aws_subnet.public.id   # référence scalaire → erreur en contexte for_each
  ...
}

# Corrigé
resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id = each.value.id   # each.value est l'objet aws_subnet courant
  ...
}
```

Quand une ressource est créée avec `for_each`, elle n'a plus d'attributs directs — `aws_subnet.public.id` est invalide. Il faut passer par `each.value` dans l'itération ou `aws_subnet.public["key"].id` depuis l'extérieur.

---

### Bug 4 — Attribut `cidr_block` au lieu de `cidr_blocks`

**Fichier :** `main.tf` — `aws_security_group.bastion`

```hcl
# Cassé
ingress {
  from_port  = 22
  to_port    = 22
  protocol   = "tcp"
  cidr_block = var.allowed_ssh_cidrs   # attribut inexistant sur ingress inline
}

# Corrigé
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = var.allowed_ssh_cidrs   # pluriel, accepte list(string)
}
```

`cidr_block` (singulier) est l'attribut de `aws_vpc` et `aws_subnet`. Sur les règles ingress/egress, c'est `cidr_blocks` (pluriel, `list(string)`). Erreur classique à cause de l'incohérence de nommage dans le provider AWS.

---

## Scénario 2 — Stack ECS Fargate + Secrets Manager

5 bugs : 3 bloquants ou silencieusement incorrects, 1 qui fera planter l'app au runtime, 1 de sécurité.

---

### Bug 1 — Mauvaise référence au log group dans l'IAM policy

**Fichier :** `iam.tf` — `data.aws_iam_policy_document.ecs_task`

```hcl
# Cassé
resources = ["${aws_cloudwatch_log_group.ecs.arn}:*"]
#                                          ^^^
# La ressource s'appelle "app_logs", pas "ecs"

# Corrigé
resources = ["${aws_cloudwatch_log_group.app_logs.arn}:*"]
```

Erreur détectée au plan : `Reference to undeclared resource`. Même cause que le bug 2 du scénario 1 — renommage non répercuté.

---

### Bug 2 — IAM trop permissif (bug de sécurité)

**Fichier :** `iam.tf` — `data.aws_iam_policy_document.ecs_task`

```hcl
# Cassé — accès à tous les secrets du compte
resources = ["*"]

# Corrigé — limité au secret de l'app
resources = [aws_secretsmanager_secret.db.arn]
```

Le `apply` passe, l'app fonctionne, mais le task role peut lire **tous les secrets du compte AWS**. En cas de compromission du container, c'est une escalade latérale triviale. Principe du moindre privilège : l'ARN exact du secret suffit.

---

### Bug 3 — `valueFrom` pointe sur la valeur du secret, pas sur son ARN

**Fichier :** `ecs.tf` — `aws_ecs_task_definition.app`, bloc `secrets`

```hcl
# Cassé — injecte la valeur brute du secret (string Terraform) comme ARN
secrets = [{
  name      = "DB_PASSWORD"
  valueFrom = aws_secretsmanager_secret_version.db.secret_string
}]

# Corrigé — valueFrom attend un ARN Secrets Manager ou SSM Parameter Store
secrets = [{
  name      = "DB_PASSWORD"
  valueFrom = aws_secretsmanager_secret.db.arn
}]
```

`valueFrom` dans ECS est un ARN, pas une valeur. ECS appelle lui-même Secrets Manager au démarrage du container pour récupérer le secret. Avec la valeur brute, ECS va tenter de le traiter comme un ARN invalide → le container ne démarre pas. Le `apply` passe parce que Terraform ne valide pas le contenu du JSON de `container_definitions`.

---

### Bug 4 — `launch_type = "EC2"` sur un service Fargate

**Fichier :** `ecs.tf` — `aws_ecs_service.app`

```hcl
# Cassé
launch_type = "EC2"

# Corrigé
launch_type = "FARGATE"
```

La task definition a `requires_compatibilities = ["FARGATE"]` et `network_mode = "awsvpc"`. Avec `launch_type = "EC2"`, AWS refusera le déploiement à l'apply (pas au plan) : les instances EC2 du cluster n'existent pas, et `awsvpc` sans Fargate nécessite une configuration supplémentaire. Erreur visible uniquement après le `apply`.

---

### Bug 5 — Output d'une valeur sensible sans `sensitive = true`

**Fichier :** `outputs.tf`

```hcl
# Cassé — affiche le mot de passe en clair dans le terminal et les logs CI
output "db_secret_value" {
  value = aws_secretsmanager_secret_version.db.secret_string
}

# Corrigé
output "db_secret_value" {
  value     = aws_secretsmanager_secret_version.db.secret_string
  sensitive = true
}
```

Sans `sensitive = true`, Terraform affiche la valeur en clair dans `terraform output` et dans les logs d'un pipeline CI. La variable `db_password` est marquée `sensitive`, mais ça ne se propage pas automatiquement à l'output. Depuis Terraform 0.15, un warning est émis, mais pas une erreur — ça passe.

> **Bonus :** l'output s'appelle `db_secret_value` mais expose `secret_string` (le mot de passe en clair) plutôt que `secret_arn`. C'est probablement ce que l'appelant devrait recevoir pour appeler Secrets Manager lui-même.
