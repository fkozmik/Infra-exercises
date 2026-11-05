# Nix Exercises

## Exercise 1 — Attribute sets, types and interpolation (Easy)

**Context:** You want to describe an application's configuration in a Nix attrset.

**Goal:** From the `name`, `version` and `port` values defined in the `let`, complete the `appConfig` attrset with:
- `name`, `version`, `port` (taken as-is)
- `description`: a string built by interpolation, e.g. `"tuleap v16.4 listening on port 8080"`
- `tags`: the list `[ "prod" "backend" ]`

```nix
let
  name    = "tuleap";
  version = "16.4";
  port    = 8080;
in
{
  # To complete: appConfig = { ... };
}
```

---

## Exercise 2 — Functions and attrsets with default values (Easy-Medium)

**Context:** You want a reusable function to describe system users.

**Goal:** Write a `mkUser` function taking an **attrset** as its argument:
- `name` (required)
- `shell` (optional, defaults to `"/bin/bash"`)
- `isAdmin` (optional, defaults to `false`)

It must return an attrset `{ name, shell, groups }` where `groups` is `[ "wheel" "users" ]` if `isAdmin` is `true`, and `[ "users" ]` otherwise.

Then use `mkUser` to create two users: `alice` (admin, default shell) and `bob` (non-admin, shell `/bin/zsh`).

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

## Exercise 3 — map, filter, foldl' and listToAttrs (Medium)

**Context:** You have a list of packages, each available only in certain environments. You must build the effective list for a given environment.

**Goal:** From `packages` and `env = "dev"`:
1. Filter the packages available in `env` (with `builtins.filter`)
2. Turn the result into a `name -> true` attrset with `builtins.listToAttrs` (useful for a conditional `environment.systemPackages`, or for deduplication)
3. Compute `count`, the total number of selected packages, with `builtins.foldl'`

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
  # To complete:
  # selected : name -> true attrset for the packages available in `env`
  # count    : number of selected packages
}
```

---

## Exercise 4 — Derivation and `.override` (Medium-Hard)

**Context:** You want to package a small bash script as a Nix derivation, then allow its message to be changed without duplicating the code.

**Goal:**
1. Write a `greeter` derivation with `stdenv.mkDerivation` that installs a `bin/greet` script printing `"Hello, <name>!"` (use a `GREETING_NAME` environment variable, defaulting to `"world"`, injected into the script from `installPhase`)
2. Use `.override` to create `greeterEnalean`, a variant where `GREETING_NAME = "Enalean"`

```nix
{ pkgs ? import <nixpkgs> {} }:

let
  greeter = pkgs.stdenv.mkDerivation rec {
    pname = "greeter";
    version = "1.0";
    src = ./.;

    GREETING_NAME = "world";

    # To complete: dontUnpack, installPhase (write bin/greet using $GREETING_NAME, chmod +x)
  };
in
{
  inherit greeter;
  greeterEnalean = # ... (use .override)
}
```

---

## Exercise 5 — Overlay and NixOS module with options (Hard)

**Context:** You must package a small modification of `hello` (from nixpkgs) through an **overlay**, then expose a service through a **NixOS module** with its own options.

**Part A — Overlay**

Write an overlay `final: prev: { ... }` that redefines `hello`, changing its default message via `overrideAttrs` (add a patch or a `GREETING` environment variable — your choice, but describe your approach).

**Part B — NixOS module**

Write a module exposing:
- `services.myapp.enable` (`mkEnableOption`)
- `services.myapp.port` (`mkOption`, type `types.port`, default `8080`)

And in `config`, under `mkIf cfg.enable`:
- open the port in the firewall (`networking.firewall.allowedTCPPorts`)
- declare a minimal `systemd.services.myapp` service (`ExecStart` with the port as an argument)

```nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.myapp;
in
{
  options.services.myapp = {
    # To complete: enable, port
  };

  config = mkIf cfg.enable {
    # To complete: firewall + systemd.services.myapp
  };
}
```

**Part C — Open question**

What is the difference between an **overlay** and a **NixOS module**? When do you use one rather than the other?
