# Solutions — Nix environments to debug

---

## Scenario 1 — NixOS configuration of a bastion server

5 bugs: 3 break evaluation, 1 is silent (the conditional package is never installed properly), 2 are security issues.

---

### Bug 1 — Misspelled firewall option

**File:** `configuration.nix` — `networking.firewall`

```nix
# Broken
networking.firewall.allowedTcpPorts = [ 22 80 443 ];

# Fixed
networking.firewall.allowedTCPPorts = [ 22 80 443 ];
```

The NixOS module system validates option paths strictly: `allowedTcpPorts` doesn't exist (the official option is `allowedTCPPorts`, with TCP uppercased). Evaluation error: `The option 'networking.firewall.allowedTcpPorts' does not exist`. Unlike a plain Nix attrset, an unknown key in a module is not silently ignored.

---

### Bug 2 — Reference to an undefined variable in a non-`rec` attrset

**File:** `configuration.nix` — `services.myapp.settings`

```nix
# Broken
services.myapp.settings = {
  port = 8080;
  url  = "http://localhost:${port}";   # "port" doesn't exist in this scope
};

# Fixed — option A: reference the full value
services.myapp.settings = {
  port = 8080;
  url  = "http://localhost:${toString 8080}";
};

# Fixed — option B: use rec to allow self-reference
services.myapp.settings = rec {
  port = 8080;
  url  = "http://localhost:${toString port}";
};
```

A plain `{ ... }` attrset is **not recursive**: its members can't reference each other. You need either `rec { ... }`, or a value already defined elsewhere in scope (in a `let`, for instance). On top of that, `port` is an integer — `toString` is required to interpolate it into a string, otherwise `cannot coerce an integer to a string`.

---

### Bug 3 — `mkIf` used as a list element

**File:** `configuration.nix` — `environment.systemPackages`

```nix
# Broken
environment.systemPackages = with pkgs; [
  vim
  htop
  (mkIf cfg.enableExtras curl)
];

# Fixed
environment.systemPackages = with pkgs; [
  vim
  htop
] ++ lib.optional cfg.enableExtras curl;
```

`mkIf` returns a special attrset (`{ _type = "if"; condition = ...; content = ...; }`) meant to be merged by the module system **at the level of a whole option value** (e.g. `environment.systemPackages = mkIf cond [ ... ];`). Dropped in as a plain element in the middle of a list, that marker is never resolved by standard list merging: either it breaks the list's typing (`types.listOf package` expects packages, not `_type = "if"` attrsets), or — depending on context — the element stays as-is and is never installed. `lib.optional cond value` is the right tool to conditionally add an element inside a list.

---

### Bug 4 — Root SSH login allowed and password authentication enabled (security)

**File:** `configuration.nix` — `services.openssh.settings`

```nix
# Broken
services.openssh.settings = {
  PermitRootLogin = "yes";
  PasswordAuthentication = true;
};

# Fixed
services.openssh.settings = {
  PermitRootLogin = "no";
  PasswordAuthentication = false;
};
```

Evaluation passes and the service starts, but the bastion accepts root logins and passwords — exactly what an exposed server (a bastion!) must not do. A `deploy` user with an SSH key is already defined above: that's the access path to favour. `PermitRootLogin = "prohibit-password"` is an acceptable compromise if direct root access by key is genuinely needed.

---

### Bug 5 — Cleartext secret in the Nix store (security)

**File:** `configuration.nix` — `environment.etc`

```nix
# Broken — readable by any local user
environment.etc."myapp/db-password".text = "hunter2";

# Fixed — use a runtime secret mechanism (agenix, sops-nix, or systemd credentials)
sops.secrets."myapp/db-password" = {
  owner = "myapp";
};
# or, simpler and without an external dependency:
systemd.services.myapp.serviceConfig.LoadCredential =
  [ "db-password:/run/secrets/myapp-db-password" ];
```

Anything going through `environment.etc.*.text` (and more generally anything referenced in a derivation or a NixOS configuration) ends up in `/nix/store`, which is **world-readable by design** (necessary for sharing dependencies between users). A cleartext secret there is visible to any local account, and also stays in the Git history if the file is versioned. Use a dedicated mechanism (agenix, sops-nix, systemd `LoadCredential` pointing at a file outside the store) that keeps the secret out of `/nix/store`.

---

## Scenario 2 — Flake build + devShell

5 bugs: 1 breaks evaluation, 2 are silent (the build passes but real usage breaks), 1 is a security issue, 1 breaks `nix build` with no argument.

---

### Bug 1 — `flake-utils` used without being declared as an input

**File:** `flake.nix` — `outputs`

```nix
# Broken
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.05";
};

outputs = { self, nixpkgs, ... }:
  flake-utils.lib.eachDefaultSystem (system: ...);   # flake-utils exists nowhere

# Fixed
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.05";
  flake-utils.url = "github:numtide/flake-utils";
};

outputs = { self, nixpkgs, flake-utils, ... }:
  flake-utils.lib.eachDefaultSystem (system: ...);
```

Evaluation error: `undefined variable 'flake-utils'`. Any external dependency used in `outputs` must be declared in `inputs` **and** taken as an argument of the `outputs` function — otherwise it simply isn't in scope.

---

### Bug 2 — `nativeBuildInputs` and `buildInputs` swapped

**File:** `flake.nix` — `myapp` derivation

```nix
# Broken
nativeBuildInputs = [ pkgs.openssl ];   # openssl is a library to link against, not a build tool
buildInputs = [ pkgs.pkg-config ];      # pkg-config is a build tool, not a library to link against

# Fixed
nativeBuildInputs = [ pkgs.pkg-config ];
buildInputs = [ pkgs.openssl ];
```

`nativeBuildInputs` holds the tools that run **during the build, on the host machine** (compilers, `pkg-config`, `makeWrapper`…). `buildInputs` holds the dependencies **meant for the target machine** (libraries to link against, like `openssl`). In a native build (host = target), both categories end up in the same `PATH`/`NIX_LDFLAGS` and the mistake is invisible — the build passes. As soon as you cross-compile, or another package depends on `myapp` expecting a clean dependency split, the confusion breaks the build or drags useless dependencies into the runtime closure.

---

### Bug 3 — Cleartext password in the build (security)

**File:** `flake.nix` — `buildPhase`

```nix
# Broken
buildPhase = ''
  export DB_PASSWORD="hunter2"
  make build
'';

# Fixed — pass the secret outside the store, at runtime, not at build time
buildPhase = ''
  make build
'';
# the password is injected when the service starts (systemd LoadCredential, docker secret, etc.),
# never while the package is being built
```

`hunter2` shows up in cleartext in the build logs (visible to anyone re-running `nix build` or reading `nix log`), and potentially in the produced binary if it's compiled in. A secret has no business in a `buildPhase`: the resulting derivation is a public `/nix/store/...` path, readable by every user on the system.

---

### Bug 4 — No `packages.default`

**File:** `flake.nix` — `outputs`

```nix
# Broken — nix build (with no argument) fails: no packages.<system>.default
packages.myapp = myapp;

# Fixed
packages.myapp = myapp;
packages.default = myapp;
```

`nix build .#myapp` works, but plain `nix build` (what most users and plenty of CI setups expect) looks for `packages.<system>.default` and fails with `flake output attribute 'packages.x86_64-linux.default' is missing`. Always expose a `default` when there's only one relevant package.

---

### Bug 5 — Out-of-schema overlay + infinite recursion

**File:** `flake.nix` — last block

```nix
# Broken
nixpkgs-overlay = final: prev: {
  myapp = final.myapp.overrideAttrs (old: {
    version = "1.0.1";
  });
};

# Fixed
outputs = { self, nixpkgs, flake-utils, ... }: {
  overlays.default = final: prev: {
    myapp = prev.myapp.overrideAttrs (old: {
      version = "1.0.1";
    });
  };
} // flake-utils.lib.eachDefaultSystem (system: ... );
```

Two problems at once:
1. **Out-of-schema attribute:** `nixpkgs-overlay` isn't a name the `nix` tooling recognizes — a flake overlay must live under `overlays.<name>` (usually `overlays.default`) to be discovered by `nix flake show` or consumed by another flake.
2. **Infinite recursion:** in an overlay `final: prev: { myapp = final.myapp...`, `final.myapp` refers to `myapp` **after** this overlay is applied — i.e. its own definition. Evaluating `final.myapp` in order to define `final.myapp` loops forever (`infinite recursion encountered`). You must start from `prev.myapp`, which refers to `myapp` **before** this overlay.

> **Bonus:** even once fixed, this overlay redefines `myapp` starting from `prev.myapp` — some nixpkgs package that most likely has nothing to do with the local `myapp` built above in `outputs`. If the intent is to patch **the repo's `myapp`**, no overlay is needed at all: just modify the local derivation directly.
