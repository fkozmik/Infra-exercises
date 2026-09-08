# Exercices Nix

## Exercice 1 — Attribute sets, types et interpolation (Facile)

**Contexte :** Tu veux décrire la configuration d'une application dans un attrset Nix.

**Objectif :** À partir des valeurs `name`, `version` et `port` définies dans le `let`, complète l'attrset `appConfig` avec :
- `name`, `version`, `port` (repris tels quels)
- `description` : une chaîne construite par interpolation, ex. `"tuleap v16.4 listening on port 8080"`
- `tags` : la liste `[ "prod" "backend" ]`

```nix
let
  name    = "tuleap";
  version = "16.4";
  port    = 8080;
in
{
  # Compléter : appConfig = { ... };
}
```

---

## Exercice 2 — Fonctions et attrsets avec valeurs par défaut (Facile-Moyen)

**Contexte :** Tu veux une fonction réutilisable pour décrire des utilisateurs système.

**Objectif :** Écris une fonction `mkUser` qui prend un **attrset** en argument :
- `name` (obligatoire)
- `shell` (optionnel, défaut `"/bin/bash"`)
- `isAdmin` (optionnel, défaut `false`)

Elle doit retourner un attrset `{ name, shell, groups }` où `groups` vaut `[ "wheel" "users" ]` si `isAdmin` est `true`, sinon `[ "users" ]`.

Utilise ensuite `mkUser` pour créer deux utilisateurs : `alice` (admin, shell par défaut) et `bob` (non-admin, shell `/bin/zsh`).

```nix
let
  mkUser = # ...
in
{
  alice = mkUser { name = "alice"; isAdmin = true; };
  bob   = mkUser { name = "bob"; shell = "/bin/zsh"; };
}
```

---

## Exercice 3 — map, filter, foldl' et listToAttrs (Moyen)

**Contexte :** Tu as une liste de packages, chacun disponible seulement dans certains environnements. Tu dois générer la liste effective pour un environnement donné.

**Objectif :** À partir de `packages` et de `env = "dev"` :
1. Filtre les packages disponibles dans `env` (via `builtins.filter`)
2. Transforme le résultat en attrset `nom -> true` avec `builtins.listToAttrs` (utile pour un `environment.systemPackages` conditionnel, ou pour dédupliquer)
3. Calcule `count`, le nombre total de packages retenus, avec `builtins.foldl'`

```nix
let
  packages = [
    { name = "vim";  envs = [ "dev" "prod" ]; }
    { name = "htop"; envs = [ "dev" ]; }
    { name = "gcc";  envs = [ "dev" ]; }
    { name = "nginx"; envs = [ "prod" ]; }
  ];
  env = "dev";
in
{
  # Compléter :
  # selected : attrset nom -> true pour les packages disponibles dans `env`
  # count    : nombre de packages retenus
}
```

---

## Exercice 4 — Derivation et `.override` (Moyen-Difficile)

**Contexte :** Tu veux packager un petit script bash comme une derivation Nix, puis permettre de changer son message sans dupliquer le code.

**Objectif :**
1. Écris une derivation `greeter` avec `stdenv.mkDerivation` qui installe un script `bin/greet` affichant `"Hello, <name>!"` (utilise une variable d'environnement `GREETING_NAME`, avec un défaut `"world"`, injectée dans le script via `installPhase`)
2. Utilise `.override` pour créer `greeterEnalean`, une variante où `GREETING_NAME = "Enalean"`

```nix
{ pkgs ? import <nixpkgs> {} }:

let
  greeter = pkgs.stdenv.mkDerivation rec {
    pname = "greeter";
    version = "1.0";
    src = ./.;

    GREETING_NAME = "world";

    # Compléter : dontUnpack, installPhase (écrire bin/greet avec $GREETING_NAME, chmod +x)
  };
in
{
  inherit greeter;
  greeterEnalean = # ... (utiliser .override)
}
```

---

## Exercice 5 — Overlay et module NixOS avec options (Difficile)

**Contexte :** Tu dois packager une petite modification de `hello` (nixpkgs) via un **overlay**, puis exposer un service via un **module NixOS** avec ses propres options.

**Partie A — Overlay**

Écris un overlay `final: prev: { ... }` qui redéfinit `hello` en changeant son message par défaut via `overrideAttrs` (ajoute un patch ou une variable d'environnement `GREETING`, au choix — décris ton approche).

**Partie B — Module NixOS**

Écris un module qui expose :
- `services.myapp.enable` (`mkEnableOption`)
- `services.myapp.port` (`mkOption`, type `types.port`, défaut `8080`)

Et dans `config`, avec `mkIf cfg.enable` :
- ouvre le port dans le firewall (`networking.firewall.allowedTCPPorts`)
- déclare un service `systemd.services.myapp` minimal (`ExecStart` avec le port en argument)

```nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.myapp;
in
{
  options.services.myapp = {
    # Compléter : enable, port
  };

  config = mkIf cfg.enable {
    # Compléter : firewall + systemd.services.myapp
  };
}
```

**Partie C — Question ouverte**

Quelle est la différence entre un **overlay** et un **module NixOS** ? Dans quel cas utilises-tu l'un plutôt que l'autre ?
