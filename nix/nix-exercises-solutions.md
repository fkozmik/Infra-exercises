# Solutions — Nix Exercises

## Exercise 1 — Attribute sets, types and interpolation

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

**Key points:**
- Interpolation `"${...}"` only accepts strings (or values that coerce to a string). `port` is an integer, so `toString port` is required — otherwise: `cannot coerce an integer to a string`.
- `inherit name version port;` avoids repeating `name = name;` — Nix looks these names up in the enclosing scope (here, the `let`).

---

## Exercise 2 — Functions and attrsets with default values

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

**Key points:**
- The pattern `{ name, shell ? default, isAdmin ? default }:` is an attrset argument with default values — very common in NixOS modules and derivations. Fields not listed in the pattern make the call fail, unless you add `, ...` at the end of the pattern.
- Nix is lazy and dynamically typed: nothing prevents calling `mkUser { name = "carol"; }` — `shell` and `isAdmin` simply take their defaults.

---

## Exercise 3 — map, filter, foldl' and listToAttrs

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

**Key points:**
- `builtins.listToAttrs` expects a list of `{ name = ...; value = ...; }` — it's the standard bridge from a list to an attrset, the Nix equivalent of a Terraform `for_each` turning a collection into an indexed map.
- `foldl'` (with the apostrophe) is the **strict** version of `foldl` — it evaluates the accumulator at each step instead of building a lazy thunk. On a large list, non-strict `foldl` can blow the stack; `foldl'` is the recommended default.
- A more idiomatic alternative for `count`: `builtins.length filtered`. `foldl'` is here for the sake of the exercise, but in real code prefer the most direct function.

---

## Exercise 4 — Derivation and `.override`

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

**Key points:**
- `dontUnpack = true` is required because `src = ./.;` isn't an archive to extract here — without it, `mkDerivation` runs its default `unpackPhase` and fails for lack of a recognized archive.
- `.override` only works on derivations built through a function whose arguments are captured (the standard case of a parameterized function in nixpkgs calling `mkDerivation rec { ... }`). Here, for `.override` to apply cleanly to `GREETING_NAME`, `greeter` must be the direct result of `stdenv.mkDerivation { ... }` — which it is.
- Classic trap: the `${GREETING_NAME}` inside the bash heredoc is interpreted by **Nix** (string interpolation), not by bash at runtime — so the value is frozen when the store path is built, not read dynamically from the running script's environment.

---

## Exercise 5 — Overlay and NixOS module with options

### Part A — Overlay

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

**Key points:**
- `overrideAttrs` receives the derivation's previous attrset (`old`) and returns the fields to merge on top. Preserve any existing `old.postPatch` with `(old.postPatch or "")` rather than overwriting it.
- In an overlay, `prev.hello` refers to `hello` **before** this overlay (base nixpkgs, or the result of previous overlays); `final.hello` would refer to `hello` **after** all overlays (the final fixed point). Using `final.hello.overrideAttrs` here would cause **infinite recursion**, since `final.hello` depends on the very definition being written.

### Part B — NixOS module

```nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.myapp;
in
{
  options.services.myapp = {
    enable = mkEnableOption "the myapp service";

    port = mkOption {
      type = types.port;
      default = 8080;
      description = "Port myapp listens on.";
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

**Key points:**
- Everything must sit under `mkIf cfg.enable` inside `config` — otherwise the service and the firewall rule would exist even when `services.myapp.enable = false`, against the module's intent.
- `types.port` validates that the value is an integer between 0 and 65535 — preferable to `types.int`, which would accept any integer, including negative ones.
- `networking.firewall.allowedTCPPorts` is a **list** that merges across modules (several modules can each add ports without stepping on each other) — NixOS's `types.listOf` handles that merge automatically.

### Part C — Overlay vs NixOS module

| | Overlay | NixOS module |
|---|---|---|
| Scope | Modifies/adds **packages** in the `pkgs` attrset | Modifies the **system configuration** (services, network, users…) |
| Mechanism | A `final: prev: { ... }` function applied to the nixpkgs fixed point | An `{ options, config, imports }` attrset merged by the module system |
| Used in | `nixpkgs.overlays` (NixOS) or `import nixpkgs { overlays = [...]; }` | `environment.etc`, `imports` in `configuration.nix` |
| Typical use case | Patch a package, change its version, add a custom package to `pkgs` | Enable/configure a service, expose reusable, type-validated options |

**When to prefer one over the other:**
- Overlay: whenever you touch **how a package is built** (patch, version, build dependencies).
- Module: whenever you touch **how the system is configured** (which services run, which ports are open, which users exist). A module can perfectly well reference a package coming from an overlay (`pkgs.myapp` in the example above).
