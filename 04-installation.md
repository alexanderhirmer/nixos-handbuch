---
title: "Installation"
weight: 4
---

# Installation

## Lernziele

- Du kannst NixOS manuell partitionieren (parted) und alternativ deklarativ mit disko.
- Du verstehst, was `hardware-configuration.nix` enthält und warum du sie nicht von Hand pflegst.
- Du kannst eine Installation sowohl über den Channel-Weg als auch über Flakes abschließen.
- Du kennst den Unterschied zwischen einer Proxmox-VM- und einer Proxmox-LXC-Installation – das sind zwei grundverschiedene Wege.
- Du weißt, wohin unattended Installation führt (Kapitel 14) und was Impermanence damit zu tun hat (Kapitel 16).

## Warum das wichtig ist

Hier trifft alles bisher Gelernte zum ersten Mal auf eine echte Maschine. Ein Fehler in `configuration.nix` lässt sich später mit einem Rollback beheben (Kapitel 2 und 7) – ein Fehler in der Partitionierung oder beim Bootloader nicht ohne Weiteres. Es lohnt sich, diesen ersten Schritt in einer Wegwerf-VM zu üben, bevor echte Hardware oder echte Daten im Spiel sind.

## Plattform wählen

Drei Zielplattformen kommen in diesem Kapitel vor:

- **Bare Metal / generische VM (QEMU, libvirt, VirtualBox aus dem Vorwort):** Der volle manuelle Weg – ISO booten, partitionieren, installieren. Alles Folgende in diesem Kapitel bezieht sich primär hierauf.
- **Proxmox als VM:** Technisch identisch zum generischen VM-Weg, nur dass die ISO über Proxmox statt über VirtualBox eingehängt wird. Kein eigener Abschnitt nötig.
- **Proxmox als LXC-Container:** Ein *komplett anderer* Weg – kein ISO-Boot, sondern ein fertiges Container-Image. Eigener Abschnitt weiter unten, unbedingt nicht mit "NixOS in einer Proxmox-VM" verwechseln.

## Partitionierung – manuell

Nach dem Boot von der Minimal-ISO (Vorwort) bist du als `nixos` eingeloggt, `sudo -i` gibt dir Root-Rechte. `lsblk` zeigt dir die verfügbaren Geräte.

**UEFI-Systeme (GPT), Beispiel `/dev/sda`:**

```console
# parted /dev/sda -- mklabel gpt
# parted /dev/sda -- mkpart root ext4 512MB -8GB
# parted /dev/sda -- mkpart swap linux-swap -8GB 100%
# parted /dev/sda -- mkpart ESP fat32 1MB 512MB
# parted /dev/sda -- set 3 esp on
```

**BIOS-Systeme (MBR), Beispiel `/dev/sda`:**

```console
# parted /dev/sda -- mklabel msdos
# parted /dev/sda -- mkpart primary 1MB -8GB
# parted /dev/sda -- set 1 boot on
# parted /dev/sda -- mkpart primary linux-swap -8GB 100%
```

**Formatieren und mounten** (UEFI-Variante):

```console
# mkfs.ext4 -L nixos /dev/sda1
# mkswap -L swap /dev/sda2
# swapon /dev/sda2
# mkfs.fat -F 32 -n boot /dev/sda3
# mount /dev/disk/by-label/nixos /mnt
# mkdir -p /mnt/boot
# mount -o umask=077 /dev/disk/by-label/boot /mnt/boot
```

(Quelle für beide Partitionsschemata und die Formatierungsbefehle: NixOS Manual, Abschnitt "Manual Installation".)

## Partitionierung – deklarativ mit disko

Handpartitionierung ist der einzige Schritt einer NixOS-Installation, der klassisch *nicht* deklarativ ist – jede Neuinstallation tippst du dieselben `parted`-Befehle erneut. **disko** ([nix-community/disko](https://github.com/nix-community/disko)) schließt genau diese Lücke: Du beschreibst dein Partitionsschema als Nix-Ausdruck, und ein einziger Befehl partitioniert, formatiert und mountet danach exakt danach.

Ein Beispiel für eine einfache GPT-Platte mit EFI- und Root-Partition (gekürzt aus dem offiziellen `hybrid`-Beispiel):

```nix
# disko-config.nix
{
  disko.devices = {
    disk = {
      main = {
        type = "disk";
        device = "/dev/vda";   # unbedingt an dein System anpassen, siehe lsblk
        content = {
          type = "gpt";
          partitions = {
            ESP = {
              size = "500M";
              type = "EF00";
              content = {
                type = "filesystem";
                format = "vfat";
                mountpoint = "/boot";
              };
            };
            root = {
              size = "100%";
              content = {
                type = "filesystem";
                format = "ext4";
                mountpoint = "/";
              };
            };
          };
        };
      };
    };
  };
}
```

Ausgeführt wird das Ganze mit dem Flake-basierten Runner:

```console
# nix --experimental-features "nix-command flakes" run github:nix-community/disko/latest -- --mode destroy,format,mount /tmp/disko-config.nix
```

> ⚠️ Ungeprüft/versionsabhängig: Die genauen `--mode`-Werte von disko haben sich zwischen Versionen schon verschoben (ältere Anleitungen zeigen z. B. `--mode disko` oder `--mode zap_create_mount` statt der kommagetrennten Form oben). Prüfe im Zweifel die aktuelle [Quickstart-Doku](https://github.com/nix-community/disko/blob/master/docs/quickstart.md), bevor du das eins zu eins übernimmst – das ist die Art Detail, die sich in einem community-getragenen Tool schneller ändert als im NixOS Manual selbst.

Der Vorteil zeigt sich vor allem in Kapitel 14: Mit derselben `disko-config.nix` lässt sich in Kombination mit `nixos-anywhere` eine Maschine komplett unbeaufsichtigt über SSH aufsetzen, ganz ohne USB-Stick und ohne dass jemand `parted`-Befehle abtippt.

**Warnung, die man nicht übersehen sollte:** Sowohl die manuelle als auch die disko-Variante löschen alle bestehenden Daten auf der Zielplatte. Es gibt hier kein "Undo".

## `hardware-configuration.nix` verstehen

Nach dem Partitionieren und Mounten auf `/mnt` generierst du eine erste Konfiguration:

```console
# nixos-generate-config --root /mnt
```

Das erzeugt zwei Dateien: `/mnt/etc/nixos/configuration.nix` (die *du* pflegst) und `/mnt/etc/nixos/hardware-configuration.nix` (die *nicht* du pflegst). Letztere enthält, was `nixos-generate-config` aus der aktuell erkannten Hardware und den aktuell gemounteten Dateisystemen abgeleitet hat – Festplattenlayout, Kernel-Module, CPU-Mikrocode. Sie wird bei jedem erneuten Lauf von `nixos-generate-config` überschrieben; manuelle Änderungen daran gehen also verloren, und Änderungen, die du dauerhaft haben willst, gehören stattdessen in `configuration.nix`.

## Konfiguration abschließen: Channel-Weg vs. Flake-Weg

Ab hier trennen sich die beiden Wege, die dieses Buch parallel behandelt.

**Channel-Weg** (klassisch, in den meisten Tutorials im Netz):

```console
# nixos-generate-config --root /mnt
# nano /mnt/etc/nixos/configuration.nix
# nixos-install
```

**Flake-Weg:**

```console
# nixos-generate-config --root /mnt --flake
# nano /mnt/etc/nixos/flake.nix
# nano /mnt/etc/nixos/configuration.nix
# nixos-install --flake '/mnt/etc/nixos#nixos'
```

Der Unterschied: `--flake` bei `nixos-generate-config` legt zusätzlich ein `flake.nix` an, und `nixos-install` bekommt mit `--flake 'pfad#konfigurationsname'` gesagt, welche der (potenziell mehreren) in `flake.nix` definierten Konfigurationen gemeint ist – `nixos` ist dabei nur der Name, den die generierte Vorlage vergibt, kein Fixwert. Was ein Flake eigentlich *ist* und warum das für Reproduzierbarkeit relevant ist, erklärt Kapitel 8 ausführlich; hier zählt nur die Mechanik, um überhaupt fertig zu werden.

Minimalbeispiel für den generierten Konfigurationsinhalt (Channel-Weg, UEFI):

```nix
# /mnt/etc/nixos/configuration.nix
{ config, pkgs, ... }:
{
  imports = [ ./hardware-configuration.nix ];

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "buch-vm";

  services.openssh.enable = true;

  users.users.alex = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
  };

  system.stateVersion = "26.05";
}
```

`system.stateVersion` trägt `nixos-generate-config` automatisch ein – es markiert, mit welcher NixOS-Version einige zustandsbehaftete Dienste (Datenbank-Formate und Ähnliches) ursprünglich initialisiert wurden, und wird bei einem Release-Upgrade (Kapitel 9) bewusst *nicht* automatisch mit hochgezählt.

`nixos-install` fragt dich am Ende nach dem Root-Passwort. Für unbeaufsichtigte Installationen (Kapitel 14) gibt es `--no-root-passwd`.

## Erster Boot

```console
# reboot
```

Nach dem Neustart zeigt das Bootmenü (systemd-boot oder GRUB, je nach Konfiguration) genau eine verfügbare Generation – noch nichts Spannendes, aber es ist exakt derselbe Mechanismus, der ab dem zweiten `nixos-rebuild switch` das Rollback ermöglicht (Kapitel 2, vertieft in Kapitel 7). Melde dich mit dem in `configuration.nix` deklarierten Nutzer an.

## Proxmox als VM

Nichts Neues gegenüber dem generischen VM-Weg oben: ISO als Storage-Content ("ISO Images") in Proxmox hochladen, neue VM anlegen, die ISO als CD/DVD einhängen, booten, ab hier identisch mit allem bisher Beschriebenen.

## Proxmox als LXC-Container

Das ist der Weg, bei dem die meisten Verwechslungen passieren: Proxmox-LXC-Container sind **nicht** dasselbe wie NixOS' eigene `nixos-container`-Mechanik (die kommt an anderer Stelle vor). Hier bootest du auch keine ISO – stattdessen baust du ein fertiges Container-Template und lädst es hoch.

Nixpkgs bringt dafür ein eigenes Modul mit, `virtualisation/proxmox-lxc.nix`. Eine minimale Konfiguration, die dieses Modul einbindet:

```nix
{ pkgs, modulesPath, ... }:
{
  imports = [ (modulesPath + "/virtualisation/proxmox-lxc.nix") ];
  environment.systemPackages = [ pkgs.vim ];
}
```

Gebaut wird das Template mit dem Community-Tool `nixos-generators`:

```console
$ nix run github:nix-community/nixos-generators -- --format proxmox-lxc -c ./configuration.nix
```

Das Ergebnis ist eine `.tar.xz`-Datei, die du in Proxmox unter "CT-Templates" hochlädst und darauf basierend einen Container anlegst.

> ⚠️ Bekannte Einschränkung (Quelle: NixOS-Wiki, Stand Dezember 2024): `nixos-rebuild` innerhalb eines so erzeugten Proxmox-LXC-Containers wendet Änderungen laut mehreren übereinstimmenden Community-Berichten teils nicht zuverlässig an, obwohl der Befehl scheinbar erfolgreich durchläuft. Der pragmatische Workaround ist, Änderungen in die Konfiguration einzubauen und ein neues Template zu bauen, statt sich auf Live-Rebuilds im laufenden Container zu verlassen. Ob sich das inzwischen gebessert hat, kannst du auf der aktuellen [NixOS-Wiki-Seite zu Proxmox](https://wiki.nixos.org/wiki/Proxmox_Virtual_Environment) nachsehen.
>
> Ebenfalls laut Wiki: Root-Passwort und Hostname lassen sich über die Proxmox-Web-UI bei NixOS-LXC-Containern nicht zuverlässig setzen – Netzwerkkonfiguration und SSH-Keys für Root dagegen schon. Praktikabler Weg: Root-Passwort beim ersten Boot direkt im Container setzen.

## Kurzer Ausblick: Unattended Installation

Die Kombination aus `disko` (deklarative Partitionierung, siehe oben) und `nixos-anywhere` (Provisionierung über SSH, ganz ohne physischen oder virtuellen Konsolenzugriff) macht aus allem, was in diesem Kapitel manuell passiert ist, einen einzigen Befehl von einer beliebigen anderen Maschine aus. Das ist besonders für mehrere gleichartige Server relevant und wird in Kapitel 14 vollständig behandelt.

> 💡 **Nice to know:** Bei ungewöhnlicher Hardware erkennt `nixos-generate-config` nicht immer alles automatisch – dann brauchst du unter Umständen `boot.initrd.kernelModules`, um die richtigen Kernel-Module für den Root-Dateisystem-Zugriff schon im Initrd verfügbar zu machen (NixOS Manual, Abschnitt "Manual Installation"). Für viele gängige Geräte (insbesondere Laptops und Einplatinencomputer) gibt es im Repository [NixOS/nixos-hardware](https://github.com/NixOS/nixos-hardware) fertige, getestete Profile zum Importieren, bevor du bei null anfängst.

> 💡 **Nice to know:** Wer noch einen Schritt weiter gehen will: Bei **Impermanence** wird das Root-Dateisystem bei jedem Boot komplett verworfen (z. B. per tmpfs oder ZFS-Rollback), und nur explizit als "zu behaltend" markierte Pfade überleben einen Neustart. Das Community-Projekt dazu heißt schlicht [impermanence](https://github.com/nix-community/impermanence). Volle Einordnung – inklusive der Frage, wann sich der zusätzliche Aufwand lohnt – folgt in Kapitel 16.

## Typische Fehler

**1. `boot.loader.grub.device` auf einem BIOS-System vergessen:**

```
Failed assertions:
- You must set the option `boot.loader.grub.device` or `boot.loader.grub.devices` to make the system bootable.
```

*Ursache:* Auf BIOS-Systemen weiß NixOS ohne diese Option nicht, auf welche Platte GRUB installiert werden soll.
*Fix:* `boot.loader.grub.device = "/dev/sda";` (oder das jeweils passende Gerät) setzen.

**2. `nixos-install` ohne vorherigen `nixos-generate-config`-Lauf (oder mit falschem `--root`):**

```
error: getting status of '/mnt/etc/nixos/hardware-configuration.nix': No such file or directory
```

*Ursache:* `configuration.nix` importiert `./hardware-configuration.nix`, die Datei existiert aber nicht, weil `nixos-generate-config --root /mnt` entweder gar nicht oder mit falschem Zielpfad gelaufen ist.
*Fix:* `nixos-generate-config --root /mnt` (erneut) ausführen, dann `nixos-install` wiederholen.

## Übung

1. Installiere NixOS in deiner Buch-VM aus dem Vorwort komplett manuell: partitionieren, formatieren, mounten, `nixos-generate-config`, `configuration.nix` anpassen (Hostname, ein Nutzer, `services.openssh.enable`), `nixos-install`, neu starten, einloggen.
2. Wiederhole dieselbe Installation in einer zweiten, frischen VM – diesmal über den Flake-Weg. Vergleiche danach `/etc/nixos` auf beiden Maschinen: Was ist zusätzlich da?

**Lösungsskizze:**

Zu 1 und 2: Folge den Befehlsblöcken oben in Reihenfolge. Der Unterschied zwischen beiden Läufen beschränkt sich im Wesentlichen auf eine zusätzliche `flake.nix`-Datei und die `--flake`-Flags bei `nixos-generate-config` und `nixos-install`; `hardware-configuration.nix` und der grobe Ablauf bleiben identisch.

## Zusammenfassung

- Partitionierung ist der einzige klassisch nicht-deklarative Schritt – disko schließt diese Lücke, mit versionsabhängigen CLI-Details.
- `hardware-configuration.nix` wird generiert, nicht von Hand gepflegt; eigene Anpassungen gehören in `configuration.nix`.
- Channel-Weg und Flake-Weg unterscheiden sich installationstechnisch nur um wenige Flags (`--flake` bei Generierung und Install) – die konzeptionellen Unterschiede folgen in Kapitel 8.
- Proxmox als VM ist nichts Neues; Proxmox als LXC-Container ist ein komplett anderer, imagebasierter Weg mit bekannten Einschränkungen beim Live-Rebuild.
- `boot.loader.grub.device` auf BIOS-Systemen und ein vorheriger `nixos-generate-config`-Lauf sind die häufigsten Stolperfallen.
- disko + nixos-anywhere ergeben zusammen unbeaufsichtigte Installation (Kapitel 14); Impermanence geht noch einen Schritt weiter (Kapitel 16).

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `parted`-Partitionsschemata (UEFI/BIOS), `mkfs.*`, `mount`, `nixos-generate-config`, `nixos-install` (inkl. `--flake`) | [NixOS Manual – Installation](https://nixos.org/manual/nixos/stable/) |
| `boot.loader.grub.device`-Assertion | NixOS-Modulsystem (GRUB-Modul), sinngemäß aus NixOS Manual |
| disko, Beispielkonfiguration und CLI | [nix-community/disko](https://github.com/nix-community/disko), [Quickstart-Doku](https://github.com/nix-community/disko/blob/master/docs/quickstart.md) |
| `proxmox-lxc.nix`-Modul, `nixos-generators --format proxmox-lxc` | [nixpkgs-Quellcode](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/virtualisation/proxmox-lxc.nix), [NixOS-Wiki – Proxmox Virtual Environment](https://wiki.nixos.org/wiki/Proxmox_Virtual_Environment) |
| `NixOS/nixos-hardware` | https://github.com/NixOS/nixos-hardware |
| `impermanence` | https://github.com/nix-community/impermanence |
