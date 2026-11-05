# DevOps Exercises

Terraform and Nix exercises with commented solutions.

Two formats:

- **Exercises** — code to complete, in increasing difficulty (easy → hard).
- **Debug** — full environments that look functional but contain bugs to find and fix (broken evaluation, incorrect behaviour, security holes).

## Contents

```
terraform/
  terraform-exercises.md             5 exercises — variables, locals, for_each, modules, data sources
  terraform-exercises-solutions.md
  terraform-debug-exercises.md       2 scenarios — multi-AZ VPC, ECS Fargate + Secrets Manager
  terraform-debug-solutions.md

nix/
  nix-exercises.md                   5 exercises — attrsets, functions, lists, derivations, overlays
  nix-exercises-solutions.md
  nix-debug-exercises.md             2 scenarios — NixOS bastion config, flake build + devShell
  nix-debug-solutions.md
```

## Usage

Work through an exercise file, then compare with its `-solutions` counterpart.
Solutions explain the *why*, not just the corrected code.
