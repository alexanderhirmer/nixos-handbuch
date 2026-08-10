---
title: "Container-Laufzeit (Podman)"
weight: 10
---

# Schritt 10: `virtualisation.podman` läuft, mit offen benannten Grenzen des unprivilegierten LXC

## Ziel

`virtualisation.podman` ist aktiv, sodass der Runner (ab Schritt 11) Job-Container starten kann. Die dafür nötigen Proxmox-LXC-Features sind gesetzt, und die Einschränkungen eines *unprivilegierten* Containers dabei sind offen benannt statt glattgebügelt.

## Voraussetzung

Schritte 1–9 sind abgeschlossen: Baseline vollständig (Nutzer, Sudo, SSH auf `<ssh-port>`, fail2ban, LDAP, Firewall). `modules/runner/` existiert noch nicht.

## Durchführung

**1. Proxmox-Host: LXC-Features freischalten.** Ein Container-Laufzeit-Prozess *innerhalb* eines LXC-Containers ist selbst wieder ein genesteter Container – Proxmox muss das explizit erlauben:

```console
$ pct set <vmid> --features nesting=1,keyctl=1,fuse=1
$ pct reboot <vmid>
```

*GUI-Äquivalent (Stand PVE 8.x/9.x):* CT `<vmid>` → "Options" → Zeile "Features" doppelklicken → Haken bei "Nesting", "Keyctl" und "FUSE" setzen.

> ⚠️ Ungeprüft: `pct.conf`s Options-Referenz war unter `pve.proxmox.com` aus dieser Umgebung nicht erreichbar (bekannte Einschränkung, siehe Fußnote in `02-flake-grundgeruest.md`). Die drei Flags sind über mehrere unabhängige Community-Quellen deckungsgleich belegt: `nesting` erlaubt dem Container, selbst wieder Container-Namespaces/Mounts aufzuspannen (sonst bleibt u. a. `/sys/fs/cgroup` beim Anlegen einer eigenen Cgroup read-only); `keyctl` gibt – nur für unprivilegierte Container relevant – den sonst per Seccomp blockierten `keyctl()`-Syscall frei, den Container-Runtimes für Kernel-Keyring-Operationen brauchen; `fuse` macht `/dev/fuse` im Gast verfügbar (Default `0`). Die GUI-Pfadangabe ist ebenfalls nicht gegen die Proxmox-Doku selbst geprüft.

**2. Warum trotzdem `fuse-overlayfs` statt des nativen Overlay-Treibers:** Podmans Storage-Layer (`containers/storage`) dokumentiert selbst, dass overlayfs-Mounts *ohne* `CAP_SYS_ADMIN` von vielen Kerneln verweigert werden und ein `mount_program` "also required on systems where the underlying storage is btrfs, aufs, zfs, overlay, or ecryptfs based file systems" ist<sup>1</sup> – und genau danach sieht ein LXC-Rootfs auf typischem Proxmox-Storage (ZFS, oder ein wiederum overlay-basierter Directory-Storage) aus. Root *innerhalb* eines unprivilegierten Containers ist nicht der echte Host-Root und hält `CAP_SYS_ADMIN` nur im eigenen, genesteten User-Namespace – ob der Kernel daraus resultierende Overlay-Mounts erlaubt, hängt von Kernelversion und Proxmox-Storage-Backend ab. `container-runtime.nix` setzt `fuse-overlayfs` deshalb *vorsorglich*, nicht erst als Reparatur nach einem Fehlschlag.

**3. Dateien anlegen** (siehe unten), `default.nix` importieren, `configuration.nix` erweitern, testweise bauen, dann aktivieren:

```console
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

**4. Smoke-Test** – der eigentliche Beweis, dass Nesting funktioniert:

```console
$ pct exec <vmid> -- podman run --rm docker.io/library/hello-world
```

## Dateien

`<repo-root>/modules/runner/container-runtime.nix` (neu):

```nix
# <repo-root>/modules/runner/container-runtime.nix
{ pkgs, ... }:
{
  virtualisation.podman = {
    enable = true;
    # Erlaubt Job-Containern desselben Workflows (Actions-"services:"-Container),
    # sich per Name statt nur per IP zu erreichen. Öffnet dafür laut
    # nixpkgs nur UDP/53 auf dem eigenen podman0-Interface, nicht global –
    # Schritt 9s "nur Port <ssh-port> eingehend" bleibt unangetastet.
    defaultNetwork.settings.dns_enabled = true;
  };

  # Vorsorglich fuse-overlayfs statt des nativen Overlay-Treibers, siehe
  # Begründung oben (Durchführung, Punkt 2).
  virtualisation.containers.storage.settings.storage.options.mount_program =
    "${pkgs.fuse-overlayfs}/bin/fuse-overlayfs";
}
```

```nix
# <repo-root>/modules/runner/default.nix
{
  imports = [
    ./container-runtime.nix
  ];
}
```

```diff
--- a/hosts/<hostname>/configuration.nix
+++ b/hosts/<hostname>/configuration.nix
@@
   imports = [
     (modulesPath + "/virtualisation/proxmox-lxc.nix")
     ../../modules/baseline
+    ../../modules/runner
   ];
```

Bewusst **kein** `dockerCompat` und **kein** `dockerSocket.enable`: Der Forgejo-Runner ruft nie ein `docker`-Binary auf, sondern spricht die Docker-kompatible API direkt über einen Socket – und genau den verdrahtet `services.gitea-actions-runner` in Schritt 11 automatisch auf Podmans eigenen Socket (`unix:///run/podman/podman.sock`), sobald `virtualisation.podman.enable` steht. Beide Podman-Optionen blieben hier ungenutzter Ballast.

**Warum Podman statt Docker:** Kein dauerhaft laufender, privilegierter Root-Daemon (Podmans Systemd-Socket wird bei Bedarf aktiviert, `dockerd` liefe permanent); der Zugriff über die `podman`-Gruppe ist dabei sicherheitstechnisch nicht besser als Dockers `docker`-Gruppe (beide sind laut Options-Beschreibung faktisch root-äquivalent), aber ohne Daemon ist die Angriffsfläche kleiner. Wichtiger für dieses Projekt: `services.gitea-actions-runner` verdrahtet `DOCKER_HOST` und die nötige `SupplementaryGroups`-Mitgliedschaft für Podman bereits mit, ganz ohne Zusatzkonfiguration – siehe Nixpkgs-Quellcode<sup>2</sup>.

## Prüfen

- `nixos-rebuild build`/`switch` laufen ohne Fehler durch (auf Nix-Ebene unabhängig davon, ob Nesting im Kernel wirklich funktioniert – dafür definiert `services.gitea-actions-runner` noch keine Instanz, siehe Schritt 11).
- `pct exec <vmid> -- systemctl is-active podman.socket` meldet `active`.
- Der Smoke-Test aus Durchführung, Punkt 4, muss "Hello from Docker!" (Podman gibt den Image-eigenen Text unverändert aus) ohne Cgroup- oder Mount-Fehler ausgeben – das ist der eigentliche Nachweis, dass Nesting greift.

## Wenn's schiefgeht

**`Error: OCI runtime error: crun: clone: Invalid argument`** oder ein `runc create failed: … unable to apply cgroup configuration: mkdir /sys/fs/cgroup/…: read-only file system` (genau dieses Fehlerbild ist für genestete, unprivilegierte Container-Szenarien real dokumentiert, wenn auch für NixOS' eigenen `nixos-container`-Mechanismus statt für Proxmox-LXC<sup>3</sup>): `nesting=1` fehlt oder der Container wurde nach `pct set` nicht neu gestartet – Features werden nur beim Start des Containers angewendet.

**Podman meldet einen Mount-/Permission-Fehler beim Anlegen des Overlay-Layers:** `fuse=1` fehlt (kein `/dev/fuse` im Gast) oder `mount_program` wurde aus `container-runtime.nix` entfernt. Mit `pct exec <vmid> -- ls -l /dev/fuse` prüfen, ob das Device überhaupt sichtbar ist.

**`podman run` hängt beim Pull ohne Fehlermeldung:** Meist DNS/Netzwerk, nicht Podman selbst – `pct exec <vmid> -- getent hosts docker.io` gegenprüfen; Schritt 9 lässt ausgehenden Verkehr ohnehin uneingeschränkt.

## Rückweg

`./container-runtime.nix` aus `modules/runner/default.nix` entfernen (oder die ganze `../../modules/runner`-Zeile aus `configuration.nix`) und rebuilden. Auf dem Proxmox-Host `pct set <vmid> --features nesting=0,keyctl=0,fuse=0` setzt die LXC-Features zurück; ein `pct reboot <vmid>` ist dafür ebenfalls nötig.

## Querverweis

Container-Laufzeiten (Podman/Docker) kommen in Teil I nicht vor – reines Projekt-II-Terrain. Direkt relevant ist aber Kapitel 4 ("Installation"), Abschnitt Proxmox-LXC: `boot.isContainer = true;` und die dort erklärten grundsätzlichen Grenzen von LXC gegenüber echten VMs.

---

<sup>1</sup> Quelle: `containers/storage`-Quellcode, `docs/containers-storage.conf.5.md` (Branch `main`), Abschnitt "STORAGE OPTIONS FOR OVERLAY TABLE", Option `mount_program` – verbatim geladen über `raw.githubusercontent.com`. https://github.com/containers/storage/blob/main/docs/containers-storage.conf.5.md

<sup>2</sup> Quelle: Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (Branch `release-26.05`): `environment … // optionalAttrs wantsPodman { DOCKER_HOST = "unix:///run/podman/podman.sock"; }` sowie `serviceConfig.SupplementaryGroups = … ++ optionals wantsPodman [ "podman" ];`. https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/continuous-integration/gitea-actions-runner.nix

<sup>3</sup> Quelle: NixOS Discourse, "Podman/docker in nixos container (ideally in unprivileged one)?", Beiträge von pshirshov (1. Nov. 2022) und ndreas (10. Nov. 2022 / 8. Apr. 2023) – behandelt NixOS' eigenen `nixos-container`/systemd-nspawn-Mechanismus, nicht Proxmox-LXC direkt; als Beleg dafür angeführt, dass die Fehlerklasse (r/o Cgroup-Dateisystem in genesteten unprivilegierten Containern) real und dokumentiert ist, nicht dass sie 1:1 auf Proxmox zutrifft. https://discourse.nixos.org/t/podman-docker-in-nixos-container-ideally-in-unprivileged-one/22909
