---
title: "Container anlegen"
weight: 1
---

# Schritt 1: Der Container wird deklarativ angelegt

## Ziel

Ein minimales NixOS-Proxmox-LXC-Template ist gebaut, auf den Proxmox-Host hochgeladen, und Container `<vmid>` ist daraus über ein versioniertes Skript angelegt, gestartet und per `pct enter` erreichbar.

## Voraussetzung

Schritt 0 (Übersicht/Platzhaltertabelle) ist abgeschlossen. Ein Rechner mit Nix und aktivierten Flakes zum Bauen des Templates steht bereit (muss nicht der Proxmox-Host selbst sein). Root- bzw. SSH-Zugriff auf den Proxmox-Host ist vorhanden.

## Durchführung

**1. Minimale Bootstrap-Konfiguration.** Bewusst nur das Nötigste – der volle Funktionsumfang kommt in Schritt 2 per `nixos-rebuild` von innen, nicht durch erneutes Bauen des Templates:

```nix
# <repo-root>/hosts/<hostname>/bootstrap.nix
{ modulesPath, ... }:
{
  imports = [ (modulesPath + "/virtualisation/proxmox-lxc.nix") ];
}
```

**2. Template bauen** (Mechanismus aus Teil I, Kapitel 4):

```console
$ nix run github:nix-community/nixos-generators -- \
    --format proxmox-lxc -c <repo-root>/hosts/<hostname>/bootstrap.nix \
    -o <repo-root>/hosts/<hostname>/result
```

**3. Hochladen.** Ein selbst gebautes Template gehört *nicht* zu den offiziellen, per `pveam` verwaltbaren Vorlagen – es wird direkt ins Template-Verzeichnis des gewünschten Storage kopiert:

```console
$ scp <repo-root>/hosts/<hostname>/result/tarball/*.tar.xz \
    root@<proxmox-host>:/var/lib/vz/template/cache/nixos-<hostname>-bootstrap.tar.xz
```

*GUI-Äquivalent (Menüpfad Stand PVE 8.x/9.x):* Datacenter → `<Node>` → Storage `local` → Reiter "CT Templates" → Upload.

**4. Prüfen, dass Proxmox das Template sieht:**

```console
$ pvesm list local --content vztmpl
```

**5. Container anlegen** – die Parameter stehen fest in einem Skript, nichts wird frei getippt:

```bash
#!/usr/bin/env bash
# <repo-root>/hosts/<hostname>/create-container.sh
set -euo pipefail
pct create <vmid> local:vztmpl/nixos-<hostname>-bootstrap.tar.xz \
  --hostname <hostname> \
  --cores 1 \
  --memory 1024 \
  --rootfs <pve-storage>:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --ostype unmanaged \
  --onboot 1
pct start <vmid>
```

`--ostype unmanaged` ist wichtig: Proxmox kennt NixOS nicht als Betriebssystemtyp und würde bei einem bekannten Typ versuchen, Netzwerk/Hostname direkt in Gastdateien zu schreiben, die es unter NixOS so nicht gibt. Die feste IP aus der Platzhaltertabelle kommt in Schritt 2 über NixOS' eigene, deklarative Netzwerkkonfiguration – nicht über Proxmox' Injection.

*GUI-Äquivalent:* "Create CT" (oben rechts) → Assistent durchklicken, Werte wie oben; das Skript ist hier trotzdem die maßgebliche Fassung, da es versioniert und wiederholbar ist.

```console
$ chmod +x <repo-root>/hosts/<hostname>/create-container.sh
$ <repo-root>/hosts/<hostname>/create-container.sh
```

## Dateien

- `<repo-root>/hosts/<hostname>/bootstrap.nix` – neu, siehe oben
- `<repo-root>/hosts/<hostname>/create-container.sh` – neu, siehe oben

## Prüfen

- `pct status <vmid>` meldet den Status `running`.
- `pct enter <vmid>` liefert eine Root-Shell im Container; `nixos-version` darin gibt eine gültige NixOS-Versionszeile aus.

## Wenn's schiefgeht

**"unable to create CT – no such logical volume" o. Ä. bei `--rootfs`:** Der Storage-Name in `<pve-storage>` existiert nicht oder unterstützt keine Container-Rootfs. Mit `pvesm status` die tatsächlich verfügbaren, Container-fähigen Storages prüfen.

**Container startet, aber `pct enter` hängt oder bricht ab:** Meist ein zu minimales oder fehlerhaftes Template (z. B. `proxmox-lxc.nix` nicht importiert). Mit `pct exec <vmid> -- <n>` prüfen, ob überhaupt Prozesse laufen; im Zweifel Template neu bauen.

**Upload schlägt mit "unable to activate storage" fehl:** Die Storage `local` hat den Inhaltstyp `vztmpl` nicht aktiviert. Fix laut Proxmox-Dokumentation: `pvesm set local --content vztmpl,rootdir,images,iso`.

## Rückweg

```console
$ pct stop <vmid>
$ pct destroy <vmid>
$ rm /var/lib/vz/template/cache/nixos-<hostname>-bootstrap.tar.xz
```

## Querverweis

Proxmox-LXC-Templates mit `proxmox-lxc.nix` und `nixos-generators`: Teil I, Kapitel 4 ("Proxmox als LXC-Container"). Dort auch die dokumentierte Einschränkung, dass `nixos-rebuild` innerhalb eines Proxmox-LXC-Containers historisch nicht immer zuverlässig war – genau deshalb baut Schritt 2 die Vollkonfiguration testweise zuerst mit `nixos-rebuild build`, bevor `switch` läuft.
