# Nix — Environnements à debugger

Chaque scénario est un fichier **fonctionnel en apparence** mais qui contient des bugs à trouver.
Certains feront planter l'évaluation (`nix eval` / `nixos-rebuild build`), d'autres passeront mais produiront un résultat incorrect, cassé au runtime, ou non sécurisé.

---

## Scénario 1 — Configuration NixOS d'un serveur bastion

L'équipe a écrit cette configuration pour un petit serveur bastion exposant une app interne.
L'évaluation ne passe pas telle quelle. Trouve et corrige tous les bugs.

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

## Scénario 2 — Flake pour build et devShell d'une app

L'équipe packagée une petite app avec un flake. `nix build .#myapp` fonctionne, mais plusieurs choses ne se comportent pas comme prévu — et il y a un problème de sécurité non détecté.

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

**Indice :** teste mentalement (ou réellement) `nix build` sans argument, `nix flake check`, et demande-toi ce qui se passe si `myapp` est utilisé comme dépendance d'un autre package cross-compilé.
