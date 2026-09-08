# Solutions — Environnements Nix à debugger

---

## Scénario 1 — Configuration NixOS d'un serveur bastion

5 bugs : 3 bloquent l'évaluation, 1 est silencieux (le paquet conditionnel n'est jamais installé correctement), 2 sont des problèmes de sécurité.

---

### Bug 1 — Option firewall mal orthographiée

**Fichier :** `configuration.nix` — `networking.firewall`

```nix
# Cassé
networking.firewall.allowedTcpPorts = [ 22 80 443 ];

# Corrigé
networking.firewall.allowedTCPPorts = [ 22 80 443 ];
```

Le module system NixOS valide strictement les chemins d'options : `allowedTcpPorts` n'existe pas (l'option officielle est `allowedTCPPorts`, avec TCP en majuscules). Erreur à l'évaluation : `The option 'networking.firewall.allowedTcpPorts' does not exist`. Contrairement à un attrset Nix classique, une clé inconnue dans un module n'est pas silencieusement ignorée.

---

### Bug 2 — Référence à une variable non définie dans un attrset non-`rec`

**Fichier :** `configuration.nix` — `services.myapp.settings`

```nix
# Cassé
services.myapp.settings = {
  port = 8080;
  url  = "http://localhost:${port}";   # "port" n'existe pas dans ce scope
};

# Corrigé — option A : référencer le chemin complet
services.myapp.settings = {
  port = 8080;
  url  = "http://localhost:${toString 8080}";
};

# Corrigé — option B : utiliser rec pour permettre l'auto-référence
services.myapp.settings = rec {
  port = 8080;
  url  = "http://localhost:${toString port}";
};
```

Un attrset `{ ... }` classique n'est **pas récursif** : ses membres ne peuvent pas se référencer entre eux. Il faut soit `rec { ... }`, soit référencer une valeur déjà définie ailleurs dans le scope (un `let`, par exemple). Ici en plus, `port` est un entier — il faut `toString` pour l'interpoler dans une chaîne, sinon `cannot coerce an integer to a string`.

---

### Bug 3 — `mkIf` utilisé comme élément de liste

**Fichier :** `configuration.nix` — `environment.systemPackages`

```nix
# Cassé
environment.systemPackages = with pkgs; [
  vim
  htop
  (mkIf cfg.enableExtras curl)
];

# Corrigé
environment.systemPackages = with pkgs; [
  vim
  htop
] ++ lib.optional cfg.enableExtras curl;
```

`mkIf` retourne un attrset spécial (`{ _type = "if"; condition = ...; content = ...; }`) destiné à être fusionné **au niveau d'une valeur d'option entière** par le module system (ex: `environment.systemPackages = mkIf cond [ ... ];`). Placé comme simple élément au milieu d'une liste, ce marqueur n'est jamais résolu par la fusion de listes standard : soit ça casse le typage de la liste (`types.listOf package` attend des packages, pas des attrsets `_type = "if"`), soit — selon le contexte — l'élément reste tel quel et n'est jamais installé. `lib.optional cond value` est l'outil correct pour ajouter conditionnellement un élément à l'intérieur d'une liste.

---

### Bug 4 — SSH root autorisé et authentification par mot de passe active (sécurité)

**Fichier :** `configuration.nix` — `services.openssh.settings`

```nix
# Cassé
services.openssh.settings = {
  PermitRootLogin = "yes";
  PasswordAuthentication = true;
};

# Corrigé
services.openssh.settings = {
  PermitRootLogin = "no";
  PasswordAuthentication = false;
};
```

L'évaluation passe, le service démarre, mais le bastion accepte les connexions root et les mots de passe — exactement ce qu'un serveur exposé (bastion !) ne doit pas faire. Un utilisateur `deploy` avec clé SSH est déjà défini plus haut : c'est la voie d'accès à privilégier. `PermitRootLogin = "prohibit-password"` est un compromis acceptable si l'accès root direct par clé est réellement nécessaire.

---

### Bug 5 — Secret en clair dans le Nix store (sécurité)

**Fichier :** `configuration.nix` — `environment.etc`

```nix
# Cassé — lisible par tout utilisateur local
environment.etc."myapp/db-password".text = "hunter2";

# Corrigé — utiliser un mécanisme de secret runtime (agenix, sops-nix, ou credentials systemd)
sops.secrets."myapp/db-password" = {
  owner = "myapp";
};
# ou, plus simple sans dépendance externe :
systemd.services.myapp.serviceConfig.LoadCredential =
  [ "db-password:/run/secrets/myapp-db-password" ];
```

Tout ce qui passe par `environment.etc.*.text` (ou plus généralement tout ce qui est référencé dans une derivation ou une configuration NixOS) finit dans `/nix/store`, qui est **lisible par tous les utilisateurs du système** par conception (nécessaire pour le partage de dépendances entre utilisateurs). Un secret en clair ici est visible par n'importe quel compte local, et reste en plus dans l'historique Git si ce fichier est versionné. Il faut un mécanisme dédié (agenix, sops-nix, `LoadCredential` systemd avec un fichier hors du store) qui garde le secret hors de `/nix/store`.

---

## Scénario 2 — Flake build + devShell

5 bugs : 1 bloque l'évaluation, 2 sont silencieux (passent le build mais cassent l'usage réel), 1 est un problème de sécurité, 1 casse l'usage sans argument de `nix build`.

---

### Bug 1 — `flake-utils` utilisé sans être déclaré en input

**Fichier :** `flake.nix` — `outputs`

```nix
# Cassé
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.05";
};

outputs = { self, nixpkgs, ... }:
  flake-utils.lib.eachDefaultSystem (system: ...);   # flake-utils n'existe nulle part

# Corrigé
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.05";
  flake-utils.url = "github:numtide/flake-utils";
};

outputs = { self, nixpkgs, flake-utils, ... }:
  flake-utils.lib.eachDefaultSystem (system: ...);
```

Erreur à l'évaluation : `undefined variable 'flake-utils'`. Toute dépendance externe utilisée dans `outputs` doit être déclarée dans `inputs` **et** reprise dans les arguments de la fonction `outputs`, sinon elle n'est simplement pas dans le scope.

---

### Bug 2 — `nativeBuildInputs` et `buildInputs` inversés

**Fichier :** `flake.nix` — derivation `myapp`

```nix
# Cassé
nativeBuildInputs = [ pkgs.openssl ];   # openssl est une lib à linker, pas un outil de build
buildInputs = [ pkgs.pkg-config ];      # pkg-config est un outil de build, pas une lib à linker

# Corrigé
nativeBuildInputs = [ pkgs.pkg-config ];
buildInputs = [ pkgs.openssl ];
```

`nativeBuildInputs` contient les outils qui tournent **pendant le build, sur la machine hôte** (compilateurs, `pkg-config`, `makeWrapper`...). `buildInputs` contient les dépendances **destinées à la machine cible** (bibliothèques à linker, comme `openssl`). En build natif (hôte = cible), les deux catégories finissent dans le même `PATH`/`NIX_LDFLAGS` et l'erreur ne se voit pas — le build passe. Dès qu'on cross-compile, ou qu'un autre package dépend de `myapp` en s'attendant à un déploiement propre des dépendances, la confusion casse la build ou embarque des dépendances inutiles dans la closure runtime.

---

### Bug 3 — Mot de passe en clair dans le build (sécurité)

**Fichier :** `flake.nix` — `buildPhase`

```nix
# Cassé
buildPhase = ''
  export DB_PASSWORD="hunter2"
  make build
'';

# Corrigé — passer le secret hors du store, au runtime, pas au build
buildPhase = ''
  make build
'';
# le mot de passe est injecté au démarrage du service (systemd LoadCredential, docker secret, etc.),
# jamais au moment de la construction du package
```

`hunter2` apparaît en clair dans les logs de build (visibles par quiconque relance `nix build` ou consulte `nix log`), et potentiellement dans le binaire produit s'il est compilé en dur. Un secret n'a rien à faire dans une `buildPhase` : la derivation résultante est un chemin `/nix/store/...` public, lisible par tous les utilisateurs du système.

---

### Bug 4 — Pas de `packages.default`

**Fichier :** `flake.nix` — `outputs`

```nix
# Cassé — nix build (sans argument) échoue : pas de packages.<system>.default
packages.myapp = myapp;

# Corrigé
packages.myapp = myapp;
packages.default = myapp;
```

`nix build .#myapp` fonctionne, mais `nix build` seul (l'usage attendu par la plupart des utilisateurs et par beaucoup de CI) cherche `packages.<system>.default` et échoue avec `flake output attribute 'packages.x86_64-linux.default' is missing`. Toujours exposer un `default` quand il n'y a qu'un seul package pertinent.

---

### Bug 5 — Overlay hors schéma + récursion infinie

**Fichier :** `flake.nix` — dernier bloc

```nix
# Cassé
nixpkgs-overlay = final: prev: {
  myapp = final.myapp.overrideAttrs (old: {
    version = "1.0.1";
  });
};

# Corrigé
outputs = { self, nixpkgs, flake-utils, ... }: {
  overlays.default = final: prev: {
    myapp = prev.myapp.overrideAttrs (old: {
      version = "1.0.1";
    });
  };
} // flake-utils.lib.eachDefaultSystem (system: ... );
```

Deux problèmes cumulés :
1. **Attribut hors schéma :** `nixpkgs-overlay` n'est pas un nom reconnu par les outils `nix` — un overlay de flake doit vivre sous `overlays.<nom>` (généralement `overlays.default`) pour être découvert par `nix flake show` ou consommé par un autre flake.
2. **Récursion infinie :** dans un overlay `final: prev: { myapp = final.myapp...`, `final.myapp` désigne le `myapp` **après** application de cet overlay — donc sa propre définition. Évaluer `final.myapp` pour définir `final.myapp` boucle indéfiniment (`infinite recursion encountered`). Il faut partir de `prev.myapp`, qui désigne le `myapp` **avant** cet overlay.

> **Bonus :** même corrigé, cet overlay redéfinit `myapp` en repartant de `prev.myapp` — un `hello`-like package de nixpkgs qui n'a probablement rien à voir avec le `myapp` local construit plus haut dans `outputs`. Si l'intention est de patcher **le `myapp` du dépôt**, il n'y a pas besoin d'overlay du tout : il suffit de modifier directement la derivation locale.
