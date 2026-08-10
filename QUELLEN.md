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

## Teil II, Projekt 1 – Gehärteter Forgejo-Runner-LXC

### Schritt 1

- Proxmox-Quellcode, `pve-container`, `src/PVE/LXC/Setup/Unmanaged.pm` (leere `setup_network`/`set_hostname`/`set_dns`-Methoden) – https://github.com/proxmox/pve-container/blob/master/src/PVE/LXC/Setup/Unmanaged.pm
- Nixpkgs-Quellcode, `nixos/modules/virtualisation/proxmox-lxc.nix` (Branch `release-26.05`; Default-Verhalten von `manageNetwork`/`manageHostName`, auch Grundlage für Schritt 2 und 10) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/virtualisation/proxmox-lxc.nix

### Schritt 2

- Proxmox-Quellcode, `pve-container`, `src/PVE/CLI/pct.pm` (Kommandotabelle für `push`/`pull`, Einschränkung "can only push files to a running CT") – https://github.com/proxmox/pve-container/blob/master/src/PVE/CLI/pct.pm
- Nixpkgs-Quellcode, `nixos/modules/tasks/network-interfaces.nix` (Branch `release-26.05`; `networking.defaultGateway` als String oder Options-Set) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/tasks/network-interfaces.nix

### Schritt 3

- Nixpkgs-Quellcode, `nixos/modules/config/users-groups.nix` (Branch `release-26.05`; `allowsLogin`-Funktion, `hashedPassword`-Beschreibung, Lockout-Assertion; auch Grundlage für Schritt 5 und 8) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/config/users-groups.nix
- Nixpkgs-Quellcode, `nixos/modules/config/update-users-groups.pl` (Branch `release-26.05`; `/etc/shadow`-Erzeugung, `"!"`-Default bei `mutableUsers = false`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/config/update-users-groups.pl

### Schritt 4

- Nixpkgs-Quellcode, `nixos/modules/security/sudo.nix` (Branch `release-26.05`; Default-Regeln für `root`/`wheel`, `wheelNeedsPassword`, `extraRules`-Beschreibung, Rendering nach `/etc/sudoers`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/security/sudo.nix

### Schritt 5

- Nixpkgs-Quellcode, `nixos/modules/services/networking/ssh/sshd.nix` (Branch `release-26.05`; `ports`, `openFirewall`, `settings.*`, `extraConfig`-Reihenfolge, `Type = "notify-reload"`; auch Grundlage für Schritt 7–9) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/networking/ssh/sshd.nix
- GitHub-Issue NixOS/nixpkgs #12867 ("openssh: Toggling PasswordAuthentication via Match config doesn't work.") – https://github.com/NixOS/nixpkgs/issues/12867
- GitHub-Issue NixOS/nixpkgs #18503 ("NixOS sshd_config and/or PAM(?) breaks this config") – https://github.com/NixOS/nixpkgs/issues/18503
- GitHub-Issue NixOS/nixpkgs #12265 (`extraConfig`/`Match`-Reihenfolge) – https://github.com/NixOS/nixpkgs/issues/12265

### Schritt 6

- Nixpkgs-Quellcode, `nixos/modules/config/networking.nix` (Branch `release-26.05`; `/etc/services`-Symlink auf `pkgs.iana-etc`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/config/networking.nix
- Nixpkgs-Quellcode, `pkgs/by-name/ia/iana-etc/package.nix` (Branch `release-26.05`; Herkunft der Portdatenbank aus `Mic92/iana-etc`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/pkgs/by-name/ia/iana-etc/package.nix
- Nixpkgs-Quellcode, `nixos/modules/system/etc/etc.nix` (Branch `release-26.05`; vollständiger Neuaufbau von `/etc` bei jeder Aktivierung) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/system/etc/etc.nix

### Schritt 7

- Nixpkgs-Quellcode, `nixos/modules/services/security/fail2ban.nix` (Branch `release-26.05`; vorkonfiguriertes `sshd`-Jail, `backend = "systemd"`, `ignoreip`, `bantime-increment`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/security/fail2ban.nix
- NixOS-Discourse, "Fail2ban is not working for sshd with systemd backend" – https://discourse.nixos.org/t/fail2ban-is-not-working-for-sshd-with-systemd-backend/48972

### Schritt 8

- Nixpkgs-Quellcode, `nixos/modules/services/misc/sssd.nix` (Branch `release-26.05`; `system.nssModules`/`system.nssDatabases`, `environmentFile`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/misc/sssd.nix
- Nixpkgs-Quellcode, `nixos/modules/security/pam.nix` (Branch `release-26.05`; automatischer `pam_sss`-Eintrag, `makeHomeDir`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/security/pam.nix
- SSSD-Upstream-Quellcode, `src/man/sssd.conf.5.xml` (Optionen `id_provider`, `auth_provider`, `enumerate`, `cache_credentials`) – https://github.com/SSSD/sssd/blob/master/src/man/sssd.conf.5.xml
- SSSD-Upstream-Quellcode, `src/man/sssd-ldap.5.xml` (Optionen `ldap_uri`, `ldap_search_base`, `ldap_schema`, `ldap_default_bind_dn`) – https://github.com/SSSD/sssd/blob/master/src/man/sssd-ldap.5.xml

### Schritt 9

- Nixpkgs-Quellcode, `nixos/modules/services/networking/firewall.nix` (Branch `release-26.05`; keine Default-Restriktion für ausgehenden Verkehr) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/networking/firewall.nix
- Nixpkgs-Quellcode, `pkgs/by-name/ip/iptables/package.nix` (Branch `release-26.05`; `nftablesCompat = true`-Default) – https://github.com/NixOS/nixpkgs/blob/release-26.05/pkgs/by-name/ip/iptables/package.nix
- Community-Recherche (Proxmox-Forum, LXC-Projekt-Diskussionen) zu `CAP_NET_ADMIN` in unprivilegierten Containern und zur Proxmox-Host-Firewall (`firewall=1`, `/etc/pve/firewall/<vmid>.fw`) – nicht gegen `pve.proxmox.com` selbst verifizierbar, im Schritt als ⚠️ Ungeprüft gekennzeichnet.

### Schritt 10

- `containers/storage`-Quellcode, `docs/containers-storage.conf.5.md` (Option `mount_program`, Overlay-Einschränkungen ohne `CAP_SYS_ADMIN`) – https://github.com/containers/storage/blob/main/docs/containers-storage.conf.5.md
- Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (Branch `release-26.05`; `DOCKER_HOST`/`SupplementaryGroups` bei Podman; auch Grundlage für Schritt 11–14) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/continuous-integration/gitea-actions-runner.nix
- NixOS-Discourse, "Podman/docker in nixos container (ideally in unprivileged one)?" – https://discourse.nixos.org/t/podman-docker-in-nixos-container-ideally-in-unprivileged-one/22909
- Community-Recherche zu den `pct`-Features `nesting`/`keyctl`/`fuse` – `pve.proxmox.com` nicht erreichbar, mehrere unabhängige Sekundärquellen deckungsgleich, im Schritt als ⚠️ Ungeprüft gekennzeichnet.

### Schritt 11

- Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (Options-Deklaration, beide Assertions, Label-Logik) – siehe Schritt 10.
- Nixpkgs-Quellcode, `nixos/lib/utils.nix` (Branch `release-26.05`; Funktion `escapeSystemdPath`; auch Grundlage für Schritt 12–13) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/lib/utils.nix
- Nixpkgs-Quellcode, `lib/modules.nix` (Branch `release-26.05`; Fehlermeldung für Pflichtoptionen ohne Default) – https://github.com/NixOS/nixpkgs/blob/release-26.05/lib/modules.nix

### Schritt 12

- Nixpkgs-Quellcode, `nixos/lib/systemd-unit-options.nix` (Branch `release-26.05`; `unitOption`-Merge-Logik) – https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/lib/systemd-unit-options.nix
- Nixpkgs-Quellcode, `lib/options.nix` (Branch `release-26.05`; `mergeEqualOption`) – https://github.com/NixOS/nixpkgs/blob/release-26.05/lib/options.nix
- systemd-Quellcode, `man/systemd.exec.xml` (`EnvironmentFile=`-Semantik, Kommentarzeilen) – https://github.com/systemd/systemd/blob/main/man/systemd.exec.xml

### Schritt 13

- Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` – siehe Schritt 10.
- Nixpkgs-Quellcode, `nixos/lib/utils.nix` – siehe Schritt 11.
- systemd-Quellcode, `src/core/execute.c` (`exec_context_load_environment()` läuft vor dem Privilegien-Drop auf `DynamicUser`) – https://github.com/systemd/systemd/blob/main/src/core/execute.c
- Sekundärquellen (Blogs/Zusammenfassungen der unter `forgejo.org` nicht erreichbaren offiziellen Doku "Runner Registration") zu Registrierungs-Scopes (instanzweit/Org/Nutzer/Repo) und zur Wiederverwendbarkeit des Tokens – im Schritt als ⚠️ Ungeprüft gekennzeichnet.

### Schritt 14

- Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (`hostPackages`-Default) – siehe Schritt 10.
- act_runner-Quellcode (GitHub-Spiegel `focs-gitea/act_runner`, da `gitea.com` blockiert war), `internal/pkg/config/config.example.yaml` (Label-Syntax `<name>:docker://<image>`/`<name>:host`) – https://github.com/focs-gitea/act_runner/blob/main/internal/pkg/config/config.example.yaml
- Sekundärquellen zu `.forgejo/workflows/`-Vorrang, `DEFAULT_ACTIONS_URL`-Default und einem zitierten Forgejo-Issue (Codeberg `forgejo/forgejo`) zu `DEFAULT_ACTIONS_URL`-Fehlern – `forgejo.org`/`codeberg.org` nicht erreichbar, im Schritt als ⚠️ Ungeprüft gekennzeichnet.

### Schritt 15

- sops-nix (GitHub), Quellcode `pkgs/sops-install-secrets/main.go` (Branch `master`; `key` wirkungslos bei `format = "dotenv"`/`binary`/`ini`, `validateSopsFile`) – https://github.com/Mic92/sops-nix/blob/master/pkgs/sops-install-secrets/main.go
- sops-nix (GitHub), Modulquellcode `modules/sops/default.nix`/`modules/sops/age.nix` (Options-Beschreibung `key`, `format`-Enum, `sshKeyPaths`-Default aus `services.openssh.hostKeys`) – https://github.com/Mic92/sops-nix (Projekt bereits in Kapitel 10 gelistet)
- sops (getsops/sops)-Quellcode, `cmd/sops/main.go` (`--input-type`/`--output-type`-Flags) – https://github.com/getsops/sops/blob/main/cmd/sops/main.go
- Eigene Verifikation per `git ls-remote --tags` gegen `github.com/Mic92/sops-nix.git`: keine versionierten Release-Tags vorhanden – Begründung dafür, dass der Pin über `flake.lock` läuft, nicht über einen Tag.

`pve.proxmox.com`, `git.proxmox.com`, `forgejo.org` und `codeberg.org` waren beim Schreiben aus dieser Umgebung heraus nicht erreichbar. Wo möglich, wurden Aussagen stattdessen gegen Quellcode-Spiegel auf GitHub verifiziert (u. a. `github.com/proxmox/pve-container`, `github.com/NixOS/nixpkgs`, `github.com/Mic92/sops-nix`); wo auch das nicht möglich war – insbesondere Forgejo-UI-Texte, Menüpfade und einzelne Proxmox-Optionsdetails –, sind die betroffenen Aussagen in den jeweiligen Schritten als `⚠️ Ungeprüft` markiert.
