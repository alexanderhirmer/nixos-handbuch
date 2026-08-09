---
title: "Anhang"
weight: 17
---

# Anhang

## Cheat Sheet

**Installation**

| Befehl | Zweck |
|---|---|
| `nixos-generate-config --root /mnt [--flake]` | Erste Konfiguration generieren |
| `nixos-install [--flake '/mnt/etc/nixos#name']` | Installation abschließen |

**Alltag**

| Befehl | Zweck |
|---|---|
| `nix shell nixpkgs#paket` | Paket temporär für die aktuelle Shell |
| `nix-env -iA nixos.paket` | Paket nutzerspezifisch installieren |

**Rebuild & Rollback**

| Befehl | Zweck |
|---|---|
| `nixos-rebuild switch` | Bauen, aktivieren, Boot-Default setzen |
| `nixos-rebuild test` | Bauen, aktivieren, *kein* neuer Boot-Default |
| `nixos-rebuild boot` | Bauen, Boot-Default setzen, erst beim nächsten Reboot aktiv |
| `nixos-rebuild dry-activate` | Zeigen, was sich ändern würde – ohne es zu tun |
| `nixos-rebuild build-vm` | Konfiguration in einer QEMU-VM testen |
| `nixos-rebuild switch --rollback` | Vorherige Generation aktivieren |
| `nixos-rebuild list-generations` | Alle Generationen auflisten |
| `nixos-rebuild switch --target-host user@host` | Remote-Deploy ohne Zusatztools |

**Updates & Wartung**

| Befehl | Zweck |
|---|---|
| `nix-channel --update` / `nixos-rebuild switch --upgrade` | Channel-Update |
| `nix flake update [input]` | Flake-Inputs aktualisieren (alle bzw. gezielt) |
| `nix-collect-garbage -d` | Alte Generationen + ungenutzte Store-Pfade löschen |
| `nix-store --optimise` | Store per Hardlinks deduplizieren |

**Reproduzierbarkeit & Fehlersuche**

| Befehl | Zweck |
|---|---|
| `nix-store -q --references/--requisites/--deriver <pfad>` | Abhängigkeiten bzw. Derivation eines Store-Pfads |
| `nix repl` / `nix eval` / `nix-instantiate --eval` | Ausdrücke interaktiv bzw. gezielt auswerten |
| `nixos-rebuild build --show-trace` | Vollen Stack-Trace bei Eval-Fehlern zeigen |
| `nix log <installable>` | Volles Build-Log |
| `journalctl -u <dienst>` / `-b` / `-b -1` / `-b -p err` | Dienst-Log, aktueller/vorheriger Boot, nur Fehler |

**Mehrere Maschinen**

| Befehl | Zweck |
|---|---|
| `nix run github:nix-community/nixos-anywhere -- --flake .#host user@ip` | Bare-Metal-Erstprovisionierung |
| `colmena apply --on @tag` | Paralleles Deployment mehrerer Hosts |
| `deploy .#node` | Deploy mit automatischem Rollback bei Fehlschlag |

## Glossar (Kurzfassung)

Die vollständige, laufend gepflegte Fassung mit allen Begriffen aus jedem Kapitel liegt in `GLOSSAR.md`. Hier nur die Begriffe, die am häufigsten wiederkehren:

- **Nix Store** – unveränderlicher Ablageort (`/nix/store/<hash>-<name>`) für so gut wie alles, was Nix baut.
- **Derivation** – die Bauanleitung (`.drv`) hinter einem Store-Pfad.
- **Generation** – ein nummerierter, bootbarer Snapshot eines Systemzustands; Grundlage für Rollback.
- **GC-Root** – Ausgangspunkt, von dem aus erreichbare Store-Pfade eine Garbage Collection überleben.
- **Closure** – die vollständige transitive Hülle aller Abhängigkeiten eines Store-Pfads.
- **`options`/`config`** – die zwei Seiten eines Moduls: was es gibt vs. welcher Wert gesetzt wird.
- **`mkIf`/`mkDefault`/`mkForce`/`mkMerge`** – die vier Grundwerkzeuge zur Prioritäts- und Bedingungssteuerung im Modulsystem.
- **Channel** – rollender, nicht versionierter Zeiger auf einen Nixpkgs-Stand.
- **Flake / `flake.lock`** – deklarierte Inputs/Outputs plus exakt fixierte Revisionen; der reproduzierbarere Nachfolger der Channels.
- **Home Manager** – eigenständiges Projekt für deklarative Nutzer-Konfiguration, kein Teil von NixOS.
- **Impermanence** – Root-Dateisystem wird bei jedem Boot verworfen; nur explizit gelistete Pfade überleben.
- **sops-nix / agenix** – Werkzeuge, um Secrets verschlüsselt im Repo zu halten und erst zur Laufzeit außerhalb des Stores zu entschlüsseln.
- **disko / nixos-anywhere** – deklarative Partitionierung bzw. unbeaufsichtigte Erstinstallation über SSH.
- **colmena / deploy-rs** – Werkzeuge für paralleles bzw. rollback-gesichertes Deployment auf mehrere Maschinen.

## Index typischer Fehlermeldungen

Kurzbeschreibung → Kapitel, in dem Ursache und Fix stehen.

| Fehlermeldung (Ausschnitt) | Kapitel |
|---|---|
| `VT-x is not available (VERR_VMX_NO_VMX)` | 0 |
| `sha256sum: WARNING: … did NOT match` | 0 |
| `bash: apt: command not found` | 1 |
| `E212: Can't open file for writing` (Vim) | 1 |
| `touch: cannot touch '/nix/store/…': Permission denied` | 2 |
| `./hello: No such file or directory` (Binary ohne Closure) | 2 |
| `error: syntax error, unexpected …, expecting ';'` | 3 |
| `error: undefined variable 'a'` (fehlendes `rec`) | 3 |
| `Failed assertions: … boot.loader.grub.device …` | 4 |
| `error: getting status of '…hardware-configuration.nix': No such file or directory` | 4 |
| `The option '…' … does not exist.` | 5, 13, 15 |
| `The unique option '…' is defined multiple times …` | 5 |
| `ssh: connect to host … port 22: Connection refused` | 6 |
| `useradd: cannot lock /etc/passwd; try again later.` | 6 |
| `error: getting status of '/etc/nixos/configuration.nix': No such file or directory` (`--flake` vergessen) | 7 |
| `warning: error(s) occurred while switching to the new configuration` | 7 |
| `warning: Git tree '…' is dirty` | 8 |
| `error: undefined variable 'home-manager'` (Flake-Output) | 8 |
| LUKS + systemd-Stage-1: Boot hängt in "A start job is running" | 9 |
| `No space left on device` (volle `/boot`) | 9 |
| `Failed to get the data key required to decrypt the SOPS file.` | 10 |
| `age: error: no identity matched any of the recipients` | 10 |
| `(stack trace truncated; use '--show-trace' …)` | 11 |
| `error: infinite recursion encountered at …` | 11 |
| `… has an unfree license (…), refusing to evaluate.` | 12 |
| `… is marked as insecure, refusing to evaluate.` | 12 |
| `Failed assertions: … home.stateVersion …` | 13 |
| `Permission denied (publickey).` | 14 |
| `sudo: a password is required` | 14 |
| `libGL error: failed to load driver` | 15 |
| `FATAL: data directory "…" does not exist` | 16 |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | 16 |

## Quellenverzeichnis

Konsolidiert aus `QUELLEN.md`, dedupliziert und nach Kategorie sortiert.

**Primärquellen (offiziell)**

- NixOS Manual – https://nixos.org/manual/nixos/stable/
- NixOS Manual, Release Notes (Anhang B) – https://nixos.org/manual/nixos/stable/release-notes
- NixOS Download-Seite – https://nixos.org/download/
- NixOS-Blog, 26.05-"Yarara"-Release-Announcement – https://nixos.org/blog/announcements/2026/nixos-2605/
- NixOS Commercial Support – https://nixos.org/community/commercial-support/
- Nix Reference Manual (`nix-store --query`, `nix repl`, `nix store optimise` u. a.) – https://nix.dev/manual/nix/
- Nixpkgs Manual (Overriding, `lib`-Funktionen) – https://nixos.org/nixpkgs/manual/
- Nixpkgs-Quellcode (GitHub, diverse Module) – https://github.com/NixOS/nixpkgs
- Nixpkgs-Issues #527478, #323396, #295218 – https://github.com/NixOS/nixpkgs/issues

**NixOS-Wiki**

- Nix Cookbook, SSH, Locales, nixos-rebuild, Flakes, Storage optimization, Agenix, Overlays, Home Manager, Proxmox Virtual Environment, Desktop environment, KDE – https://wiki.nixos.org/

**Community-Tools & -Projekte (GitHub)**

- disko, nixos-generators, nixos-hardware, impermanence, sops-nix, home-manager, colmena, deploy-rs, nixpkgs-fmt, alejandra, ad-si/nix-companies – jeweils unter `github.com/nix-community/…` bzw. `github.com/NixOS/…`
- agenix – https://wiki.nixos.org/wiki/Agenix (NixOS-Wiki als Hauptquelle verwendet)

**Praxis-Blogs, Tutorials & Communitys**

- NixOS & Flakes Book – https://nixos-and-flakes.thiscute.world/
- nixos.asia, Convert configuration.nix to be a flake – https://nixos.asia/en/configuration-as-flake
- iampavel.dev, How to Actually Read Nix Error Messages – https://iampavel.dev/blog/how-to-read-nix-error
- Michael Stapelberg, Secret Management on NixOS with sops-nix – https://michael.stapelberg.ch/
- woile.eu, NixOS with agenix – https://woile.eu/blog/agenix.html
- Graham Christensen, Erase your darlings – https://grahamc.com/blog/erase-your-darlings/
- Elis Hirwing, NixOS: tmpfs as root – https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/
- hanckmann.com, Nixos and Erasing My Darlings – https://hanckmann.com/posts/20230104-nixos-and-erasing-my-darlings/
- b.tuxes.uk, Three Years of Ephemeral NixOS – https://b.tuxes.uk/three-years-of-ephemeral-nixos.html
- BeloutreBlog, Deploying your NixOS configurations with Colmena – https://beloutreblog.yashael.fr/
- chenlijun99/dotfiles (Praxisbeispiel) – https://github.com/chenlijun99/dotfiles
- nixos-rebuild Man-Page (mankier) – https://www.mankier.com/8/nixos-rebuild
- MyNixOS (Optionsdatenbank) – https://mynixos.com/
- Lix Systems – https://git.lix.systems/lix-project/
- Determinate Systems, Blog – https://determinate.systems/blog/

**Sonstiges**

- GNU Guix – https://guix.gnu.org
- r13y.com (Reproduzierbarkeits-Tracking Nixpkgs)
