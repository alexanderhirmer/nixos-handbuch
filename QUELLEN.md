# Quellenverzeichnis

Laufend gepflegt, primär nach Kapitel sortiert. Der Anhang im fertigen Buch kompiliert daraus das finale Verzeichnis.

## Kapitel 0 – Vorwort

- NixOS Download-Seite (ISO-Varianten) – https://nixos.org/download/
- NixOS Manual, Preface & "Installing NixOS into a VirtualBox guest" – https://nixos.org/manual/nixos/stable/
- NixOS 26.05 "Yarara" Release-Announcement (nixos.org-Blog) – https://nixos.org/blog/announcements/2026/nixos-2605/

## Kapitel 1 – Was ist NixOS?

- NixOS Manual, Installation (Beispielkonfiguration mit `services.sshd.enable`) – https://nixos.org/manual/nixos/stable/
- Curated Liste "Companies using Nix/NixOS in production" – https://github.com/ad-si/nix-companies
- NixOS Commercial Support – https://nixos.org/community/commercial-support/
- GNU Guix (verwandtes Projekt) – https://guix.gnu.org

## Kapitel 2 – Das Nix-Modell

- Nix Reference Manual, `nix-store --query` (Store-Pfad-Format, `--references`/`--requisites`/`--deriver`) – https://nix.dev/manual/nix/2.30/command-ref/nix-store/query.html
- NixOS-Wiki, Nix Cookbook (Profile, Generationen, `system-<N>-link`) – https://wiki.nixos.org/wiki/Nix_Cookbook
- nix-darwin-Projekt (`/run/current-system`) – https://github.com/nix-hackers/nix-darwin
- r13y.com – Community-Projekt zur Nachverfolgung reproduzierbarer Builds in Nixpkgs

## Kapitel 3 – Nix als Sprache

- NixOS Manual, Configuration Syntax (Attribute Sets, Listen, `let/in`, Funktionen, `with`) – https://nixos.org/manual/nixos/stable/
- NixOS Manual, Adding Custom Packages (`import <nixpkgs> { }`-Muster) – https://nixos.org/manual/nixos/stable/
- nixpkgs-fmt – https://github.com/nix-community/nixpkgs-fmt
- alejandra – https://github.com/kamadorueda/alejandra

## Kapitel 4 – Installation

- NixOS Manual, Installation (Partitionierung, Formatierung, `nixos-generate-config`, `nixos-install`, Flake-Variante) – https://nixos.org/manual/nixos/stable/
- disko – https://github.com/nix-community/disko und Quickstart-Doku (https://github.com/nix-community/disko/blob/master/docs/quickstart.md)
- Nixpkgs-Quellcode, `proxmox-lxc.nix` – https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/virtualisation/proxmox-lxc.nix
- NixOS-Wiki, Proxmox Virtual Environment (Stand Dezember 2024, Einschränkungen LXC-Rebuild) – https://wiki.nixos.org/wiki/Proxmox_Virtual_Environment
- nixos-generators (Proxmox-LXC-Template bauen) – https://github.com/nix-community/nixos-generators
- NixOS/nixos-hardware – https://github.com/NixOS/nixos-hardware
- impermanence – https://github.com/nix-community/impermanence

## Kapitel 5 – Das Modulsystem

- NixOS Manual, Configuration Syntax & Modularity (options/config, `imports`, `mkBefore`, `mkForce`, Merge-Verhalten, Fehlermeldungen im Wortlaut) – https://nixos.org/manual/nixos/stable/
- Nixpkgs Manual, `lib`-Funktionen (`mkIf`, `mkDefault`, `mkMerge`, `mkOption`, `mkEnableOption`, `lib.types`)

## Kapitel 6 – Alltagsbetrieb

- NixOS Manual, Package Management & User Management – https://nixos.org/manual/nixos/stable/
- NixOS-Wiki, SSH (`services.openssh`, `settings`, `openFirewall`) – https://wiki.nixos.org/wiki/SSH
- MyNixOS, `services.sshd.enable` (Alias-Bestätigung) – https://mynixos.com/nixpkgs/option/services.sshd.enable
- Nixpkgs-Quellcode, `nixos/modules/config/i18n.nix` (`console.keyMap`) – https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/config/i18n.nix
- NixOS-Wiki, Locales – https://wiki.nixos.org/wiki/Locales

## Kapitel 7 – nixos-rebuild im Griff

- NixOS-Wiki, nixos-rebuild (Subcommands, list-generations-Format, Rollback-Grenzen Stand Dez. 2024) – https://wiki.nixos.org/wiki/Nixos-rebuild
- nixos-rebuild Man-Page (mankier) – https://www.mankier.com/8/nixos-rebuild
- NixOS Manual, switch-to-configuration/Profile-Mechanik – https://nixos.org/manual/nixos/stable/
- chenlijun99/dotfiles (Praxisbeispiel `--rollback`) – https://github.com/chenlijun99/dotfiles

## Kapitel 8 – Reproduzierbarkeit

- NixOS Manual, Upgrading NixOS (Channels) – https://nixos.org/manual/nixos/stable/
- NixOS-Wiki, Flakes – https://wiki.nixos.org/wiki/Flakes
- NixOS & Flakes Book (flake.nix-Aufbau, Beispiele) – https://nixos-and-flakes.thiscute.world/
- nixos.asia, Convert configuration.nix to be a flake (Migrationspfad, flake.lock-Aufbau) – https://nixos.asia/en/configuration-as-flake

## Kapitel 9 – Updates & Wartung

- NixOS Manual, Upgrading NixOS – https://nixos.org/manual/nixos/stable/
- NixOS Manual, Release Notes (Anhang B, systemd-Stage-1, Switch Inhibitors) – https://nixos.org/manual/nixos/stable/release-notes
- Nixpkgs-Issue #527478 (LUKS + systemd-Stage-1-Bootproblem) – https://github.com/nixos/nixpkgs/issues/527478
- NixOS-Wiki, Storage optimization – https://wiki.nixos.org/wiki/Storage_optimization
- NixOS & Flakes Book, Other Useful Tips (`configurationLimit`, `nix.gc`) – https://nixos-and-flakes.thiscute.world/nixos-with-flakes/other-useful-tips
- Nix Reference Manual, `nix store optimise` – https://nix.dev/manual/nix/2.26/command-ref/new-cli/nix3-store-optimise
- MyNixOS, `nix.settings.auto-optimise-store` – https://mynixos.com/nixpkgs/option/nix.settings.auto-optimise-store

## Kapitel 10 – Secrets

- sops-nix (GitHub) – https://github.com/Mic92/sops-nix/
- Michael Stapelberg, Secret Management on NixOS with sops-nix – https://michael.stapelberg.ch/posts/2025-08-24-secret-management-with-sops-nix/
- NixOS-Wiki, Agenix – https://wiki.nixos.org/wiki/Agenix
- woile.eu, NixOS with agenix (Warnung `builtins.readFile`) – https://woile.eu/blog/agenix.html

## Kapitel 11 – Fehlersuche

- iampavel.dev, How to Actually Read Nix Error Messages – https://iampavel.dev/blog/how-to-read-nix-error
- Nix Reference Manual, `nix repl` – https://nix.dev/manual/nix/2.30/command-ref/new-cli/nix3-repl.html
- NixOS & Flakes Book, Debugging Derivations and Nix Expressions – https://nixos-and-flakes.thiscute.world/best-practices/debugging
- NixOS-Discourse, Infinite recursion encountered by making module configurable – https://discourse.nixos.org/t/infinite-recursion-encountered-by-making-module-configurable/23508
- Lix Systems (Installer, Projektstatus) – https://git.lix.systems/lix-project/lix-installer
- Determinate Systems, Dropping upstream Nix from Determinate Nix Installer – https://determinate.systems/blog/installer-dropping-upstream/

## Kapitel 12 – Software finden & anpassen

- NixOS & Flakes Book, Overlays & Overriding – https://nixos-and-flakes.thiscute.world/nixpkgs/overlays, https://nixos-and-flakes.thiscute.world/nixpkgs/overriding
- Nixpkgs Manual, Overriding (`override`, `overrideAttrs`, `overrideDerivation`) – https://nixos.org/nixpkgs/manual/
- NixOS-Wiki, Overlays (Praxisbeispiel) – https://nixos.wiki/wiki/Overlays

## Kapitel 13 – Home Manager

- Home Manager Manual, NixOS module – https://nix-community.github.io/home-manager/installation/nixos.html
- Home Manager Manual (25.05) – https://home-manager.dev/manual/25.05/
- GitHub, nix-community/home-manager – https://github.com/nix-community/home-manager
- NixOS & Flakes Book, Getting Started with Home Manager – https://nixos-and-flakes.thiscute.world/nixos-with-flakes/start-using-home-manager
- NixOS-Wiki, Home Manager – https://wiki.nixos.org/wiki/Home_Manager

## Kapitel 14 – Mehrere Maschinen

- nixos-rebuild Man-Page (mankier) – https://www.mankier.com/8/nixos-rebuild
- NixOS & Flakes Book, Remote Deployment – https://nixos-and-flakes.thiscute.world/best-practices/remote-deployment
- deploy-rs (GitHub) – https://GitHub.com/serokell/deploy-rs
- colmena (GitHub) – https://github.com/nix-community/colmena
- Colmena-Dokumentation, Multi-Architecture-Beispiel – https://colmena.cli.rs/0.3/examples/multi-arch.html
- BeloutreBlog, Deploying your NixOS configurations with Colmena – https://beloutreblog.yashael.fr/en/articles/deploying-nixos-configurations-with-colmena/

## Kapitel 15 – Desktop (kompakt)

- Nixpkgs-Quellcode, release-26.05, gnome.nix & plasma6.nix – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/desktop-managers/
- NixOS-Wiki, Desktop environment (Kategorie) & KDE – https://wiki.nixos.org/wiki/Category:Desktop_environment, https://wiki.nixos.org/wiki/KDE
- MyNixOS, services.desktopManager.plasma6.enable – https://mynixos.com/nixpkgs/option/services.desktopManager.plasma6.enable
- Nixpkgs-Issues #323396, #295218 (hardware.graphics/NVIDIA) – https://github.com/NixOS/nixpkgs/issues/323396, https://github.com/NixOS/nixpkgs/issues/295218

## Kapitel 16 – Disaster Recovery

- nix-community/impermanence – https://github.com/nix-community/impermanence
- Graham Christensen, Erase your darlings – https://grahamc.com/blog/erase-your-darlings/
- Elis Hirwing, NixOS: tmpfs as root – https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/
- hanckmann.com, Nixos and Erasing My Darlings (Praxisbeispiel) – https://hanckmann.com/posts/20230104-nixos-and-erasing-my-darlings/
- b.tuxes.uk, Three Years of Ephemeral NixOS – https://b.tuxes.uk/three-years-of-ephemeral-nixos.html
