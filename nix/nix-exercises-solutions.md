# Solutions — Exercices Nix

## Exercice 1 — Attribute sets, types et interpolation

```nix
let
  name    = "tuleap";
  version = "16.4";
  port    = 8080;
in
{
  appConfig = {
    inherit name version port;
    description = "${name} v${version} listening on port ${toString port}";
    tags = [ "prod" "backend" ];
  };
}
```

**Points clés :**
- L'interpolation `"${...}"` n'accepte que des chaînes (ou des valeurs qui se coercent en chaîne). `port` est un entier : il faut `toString port`, sinon erreur `cannot coerce an integer to a string`.
- `inherit name version port;` évite de répéter `name = name;` — Nix va chercher ces noms dans le scope englobant (ici le `let`).

---

## Exercice 2 — Fonctions et attrsets avec valeurs par défaut

```nix
let
  mkUser = { name, shell ? "/bin/bash", isAdmin ? false }: {
    inherit name shell;
    groups = if isAdmin then [ "wheel" "users" ] else [ "users" ];
  };
in
{
  alice = mkUser { name = "alice"; isAdmin = true; };
  bob   = mkUser { name = "bob"; shell = "/bin/zsh"; };
}
```

**Points clés :**
- Le pattern `{ name, shell ? default, isAdmin ? default }:` est un argument attrset avec valeurs par défaut — très courant dans les modules NixOS et les derivations. Les champs non listés dans le pattern feront planter l'appel, sauf si on ajoute `, ...` à la fin du pattern.
- Nix est un langage paresseux et à typage dynamique : rien n'empêche d'appeler `mkUser { name = "carol"; }` sans erreur, `shell` et `isAdmin` prendront leurs défauts.

---

## Exercice 3 — map, filter, foldl' et listToAttrs

```nix
let
  packages = [
    { name = "vim";  envs = [ "dev" "prod" ]; }
    { name = "htop"; envs = [ "dev" ]; }
    { name = "gcc";  envs = [ "dev" ]; }
    { name = "nginx"; envs = [ "prod" ]; }
  ];
  env = "dev";

  filtered = builtins.filter (pkg: builtins.elem env pkg.envs) packages;
in
{
  selected = builtins.listToAttrs (map (pkg: { name = pkg.name; value = true; }) filtered);
  count    = builtins.foldl' (acc: _: acc + 1) 0 filtered;
}
```

**Points clés :**
- `builtins.listToAttrs` attend une liste de `{ name = ...; value = ...; }` — c'est le pont standard entre une liste et un attrset, l'équivalent Nix d'un `for_each` Terraform qui transforme une collection en map indexée.
- `foldl'` (avec l'apostrophe) est la version **stricte** de `foldl` — elle évalue l'accumulateur à chaque étape au lieu de construire un thunk paresseux. Sur une grande liste, `foldl` non strict peut faire exploser la pile ; `foldl'` est le choix par défaut recommandé.
- Alternative plus idiomatique pour `count` : `builtins.length filtered`. `foldl'` est ici pour l'exercice, mais dans du code réel, préfère la fonction la plus directe.

---

## Exercice 4 — Derivation et `.override`

```nix
{ pkgs ? import <nixpkgs> {} }:

let
  greeter = pkgs.stdenv.mkDerivation rec {
    pname = "greeter";
    version = "1.0";
    src = ./.;

    GREETING_NAME = "world";

    dontUnpack = true;

    installPhase = ''
      mkdir -p $out/bin
      cat > $out/bin/greet <<EOF
      #!/usr/bin/env bash
      echo "Hello, ${GREETING_NAME}!"
      EOF
      chmod +x $out/bin/greet
    '';
  };
in
{
  inherit greeter;
  greeterEnalean = greeter.override { GREETING_NAME = "Enalean"; };
}
```

**Points clés :**
- `dontUnpack = true` est nécessaire car `src = ./.;` n'est pas une archive à décompresser ici — sans lui, `mkDerivation` cherche une phase `unpackPhase` par défaut et échoue faute d'archive reconnue.
- `.override` ne fonctionne que sur les derivations construites via une fonction dont les arguments sont capturés (le cas standard de `mkDerivation rec { ... }` appelé depuis une fonction paramétrée dans nixpkgs). Ici, pour que `.override` s'applique proprement sur `GREETING_NAME`, `greeter` doit être le résultat direct de `stdenv.mkDerivation { ... }` — ce qui est le cas.
- Piège classique : le `${GREETING_NAME}` dans le heredoc bash est interprété par **Nix** (interpolation de chaîne), pas par bash au runtime — la valeur est donc figée à la construction du store path, pas lue dynamiquement depuis l'environnement du script exécuté.

---

## Exercice 5 — Overlay et module NixOS avec options

### Partie A — Overlay

```nix
final: prev: {
  hello = prev.hello.overrideAttrs (old: {
    postPatch = (old.postPatch or "") + ''
      substituteInPlace src/hello.c \
        --replace 'Hello, world!' 'Hello, Enalean!'
    '';
  });
}
```

**Points clés :**
- `overrideAttrs` reçoit l'ancien attrset de la derivation (`old`) et retourne les champs à fusionner par-dessus. On préserve `old.postPatch` existant avec `(old.postPatch or "")` plutôt que de l'écraser.
- Dans un overlay, `prev.hello` désigne le `hello` **avant** cet overlay (le nixpkgs de base ou le résultat des overlays précédents) ; `final.hello` désignerait le `hello` **après** tous les overlays (le point fixe final). Utiliser `final.hello.overrideAttrs` ici créerait une **récursion infinie** puisque `final.hello` dépend de la définition qu'on est en train d'écrire.

### Partie B — Module NixOS

```nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.myapp;
in
{
  options.services.myapp = {
    enable = mkEnableOption "le service myapp";

    port = mkOption {
      type = types.port;
      default = 8080;
      description = "Port d'écoute de myapp.";
    };
  };

  config = mkIf cfg.enable {
    networking.firewall.allowedTCPPorts = [ cfg.port ];

    systemd.services.myapp = {
      description = "myapp service";
      wantedBy = [ "multi-user.target" ];
      serviceConfig = {
        ExecStart = "${pkgs.myapp}/bin/myapp --port ${toString cfg.port}";
        Restart = "on-failure";
      };
    };
  };
}
```

**Points clés :**
- Tout doit être sous `mkIf cfg.enable` dans `config` — sinon le service et la règle firewall existeraient même quand `services.myapp.enable = false`, contrairement à l'intention du module.
- `types.port` valide que la valeur est un entier entre 0 et 65535 — préférable à `types.int` qui accepterait n'importe quel entier, y compris négatif.
- `networking.firewall.allowedTCPPorts` est une **liste** fusionnable entre modules (plusieurs modules peuvent chacun y ajouter des ports sans se marcher dessus) — c'est le système de type `types.listOf` de NixOS qui gère cette fusion automatiquement.

### Partie C — Overlay vs module NixOS

| | Overlay | Module NixOS |
|---|---|---|
| Portée | Modifie/ajoute des **packages** dans l'attrset `pkgs` | Modifie la **configuration système** (services, réseau, utilisateurs...) |
| Mécanisme | Fonction `final: prev: { ... }` appliquée au point fixe de nixpkgs | Attrset `{ options, config, imports }` fusionné par le module system |
| S'utilise dans | `nixpkgs.overlays` (NixOS) ou `import nixpkgs { overlays = [...]; }` | `environment.etc`, `imports` de `configuration.nix` |
| Cas d'usage typique | Patcher un package, changer sa version, ajouter un package custom à `pkgs` | Activer/configurer un service, exposer des options réutilisables et validées par type |

**Quand préférer l'un ou l'autre :**
- Overlay : dès que tu touches à **comment un package est construit** (patch, version, dépendances de build).
- Module : dès que tu touches à **comment le système est configuré** (quels services tournent, quels ports sont ouverts, quels utilisateurs existent). Un module peut très bien référencer un package venant d'un overlay (`pkgs.myapp` dans l'exemple ci-dessus).
