# Nix — Environments to debug

Each scenario is a file that **looks functional** but contains bugs to find.
Some will break evaluation (`nix eval` / `nixos-rebuild build`), others will pass but produce an incorrect result, broken at runtime, or insecure.

---

## Scenario 1 — NixOS configuration of a bastion server

The team wrote this configuration for a small bastion server exposing an internal app.
Evaluation doesn't pass as-is. Find and fix all the bugs.

### `configuration.nix`

```nix
{ config, pkgs, lib, ... }:

let
  cfg = config.services.myapp;
in
{
  imports = [ ./hardware-configuration.nix ];

  networking.hostName = "bastion-01";
  networking.firewall.enable = true;
  networking.firewall.allowedTcpPorts = [ 22 80 443 ];

  users.users.deploy = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... deploy@enalean"
    ];
  };

  services.openssh = {
    enable = true;
    settings = {
      PermitRootLogin = "yes";
      PasswordAuthentication = true;
    };
  };

  services.myapp = {
    enable = true;
    settings = {
      port = 8080;
      url  = "http://localhost:${port}";
    };
  };

  environment.systemPackages = with pkgs; [
    vim
    htop
    (mkIf cfg.enableExtras curl)
  ];

  environment.etc."myapp/db-password".text = "hunter2";

  system.stateVersion = "24.05";
}
```

---

## Scenario 2 — Flake for an app's build and devShell

The team packaged a small app with a flake. `nix build .#myapp` works, but several things don't behave as expected — and there is an undetected security issue.

### `flake.nix`

```nix
{
  description = "myapp - build and dev environment";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.05";
  };

  outputs = { self, nixpkgs, ... }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = import nixpkgs { inherit system; };

        myapp = pkgs.stdenv.mkDerivation {
          pname = "myapp";
          version = "1.0.0";
          src = ./.;

          nativeBuildInputs = [ pkgs.openssl ];
          buildInputs = [ pkgs.pkg-config ];

          buildPhase = ''
            export DB_PASSWORD="hunter2"
            make build
          '';

          installPhase = ''
            mkdir -p $out/bin
            cp myapp $out/bin/
          '';
        };
      in
      {
        packages.myapp = myapp;

        devShells.default = pkgs.mkShell {
          buildInputs = [ pkgs.gcc pkgs.gnumake pkgs.openssl.dev pkgs.pkg-config ];
        };
      });

  nixpkgs-overlay = final: prev: {
    myapp = final.myapp.overrideAttrs (old: {
      version = "1.0.1";
    });
  };
}
```

**Hint:** mentally (or actually) try `nix build` with no argument, `nix flake check`, and ask yourself what happens if `myapp` is used as a dependency of another cross-compiled package.
