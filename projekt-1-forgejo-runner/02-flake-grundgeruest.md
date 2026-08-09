---
title: "Flake-Grundgerüst"
weight: 2
---

# Schritt 2: Ein Flake-Grundgerüst baut und läuft mit fester IP

## Ziel

`<repo-root>/flake.nix` und `hosts/<hostname>/configuration.nix` existieren, sind im Container unter `/etc/nixos` angekommen, und `nixos-rebuild` baut daraus erfolgreich ein System mit gesetztem `hostName` und statischer IP `<ip>` – erst geprüft mit `build`, dann aktiviert mit `switch`.

## Voraussetzung

Schritt 1 ist abgeschlossen: Container `<vmid>` läuft, ist per `pct enter <vmid>` erreichbar und hängt noch per DHCP im Netz. `/etc/nixos` im Container ist leer – das Bootstrap-Template aus Schritt 1 hat nichts dorthin geschrieben, es hat nur das Image gebaut.

## Durchführung

**1. Repo-Transfer klären.** Git-Checkout direkt im Container scheidet für *diesen* Schritt aus: Das minimale Bootstrap-Template aus Schritt 1 enthält kein `pkgs.git`, und ohne aktivierte Flakes ließe sich auch kein `nix shell nixpkgs#git` nachladen. Belegbar und ohne Zusatzabhängigkeit funktioniert dagegen `pct push`, das einzelne Dateien vom Proxmox-Host in den Container kopiert, unabhängig vom Netzwerkzustand des Gasts.<sup>1</sup> Ab diesem Schritt bringt `configuration.nix` `pkgs.git` mit – künftige Änderungen lassen sich dann per `git pull` im Container holen, nur dieser erste Satz Dateien kommt per `pct push`.

**2. Dateien lokal schreiben** (siehe Abschnitt "Dateien").

**3. Zielverzeichnis im Container anlegen und Dateien pushen** (Befehle auf dem Proxmox-Host):

```console
$ pct exec <vmid> -- mkdir -p /etc/nixos/hosts/<hostname>
$ pct push <vmid> <repo-root>/flake.nix /etc/nixos/flake.nix
$ pct push <vmid> <repo-root>/hosts/<hostname>/configuration.nix \
    /etc/nixos/hosts/<hostname>/configuration.nix
```

**4. Im Container Flakes für diesen einen Aufruf aktivieren und bauen** – `nix.settings.experimental-features` aus `configuration.nix` greift erst *nach* einem erfolgreichen `switch`; für den allerersten Build braucht Nix den Hinweis noch von außen:

```console
$ pct enter <vmid>
$ echo "experimental-features = nix-command flakes" >> /etc/nix/nix.conf
$ nixos-rebuild build --flake /etc/nixos#<hostname>
```

`build` aktiviert nichts – die IP bleibt bis zum `switch` auf DHCP, ein Fehlgriff bei der statischen Konfiguration legt also noch keine Verbindung lahm. Dabei entsteht `/etc/nixos/flake.lock`, gepinnt auf `nixos-26.05`.

**5. `flake.lock` zurückholen und aktivieren:**

```console
$ pct pull <vmid> /etc/nixos/flake.lock <repo-root>/flake.lock
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

## Dateien

```nix
# <repo-root>/flake.nix
{
  description = "Infrastruktur für <hostname> (Forgejo-Runner)";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  };

  outputs = { self, nixpkgs, ... }: {
    nixosConfigurations.<hostname> = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [ ./hosts/<hostname>/configuration.nix ];
    };
  };
}
```

> 💡 **Nice to know:** Kapitel 8 lässt `system = "x86_64-linux";` meist weg, weil die generierte `hardware-configuration.nix` das schon mitbringt. Ein Proxmox-LXC-Container hat keine solche Datei – `proxmox-lxc.nix` erzeugt sie nicht –, deshalb steht `system` hier explizit im Flake.

```nix
# <repo-root>/hosts/<hostname>/configuration.nix
{ modulesPath, pkgs, ... }:
{
  imports = [ (modulesPath + "/virtualisation/proxmox-lxc.nix") ];

  # Ohne diese beiden Flags erzwingt das Modul einen leeren hostName und
  # erwartet Netzwerkdaten von Proxmox statt aus dieser Datei.
  proxmoxLXC = {
    manageNetwork = true;
    manageHostName = true;
  };

  networking = {
    hostName = "<hostname>";
    useDHCP = false;
    interfaces.eth0.ipv4.addresses = [
      { address = "<ip>"; prefixLength = 24; }
    ];
    defaultGateway = "10.20.0.1"; # Beispiel – das Gateway deines Netzes eintragen
  };

  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  environment.systemPackages = [ pkgs.git ];

  system.stateVersion = "26.05";
}
```

> ⚠️ Ungeprüft: `defaultGateway` und die Präfixlänge stehen hier als Beispiel (`/24`, Gateway `10.20.0.1`) – die Platzhaltertabelle in `00-uebersicht.md` definiert dafür keinen eigenen Platzhalter. Ermittle beides vor dem Umstieg auf `pct enter` im noch laufenden DHCP-Zustand mit `ip route show` (zeigt das aktuell zugewiesene Gateway) und trage die realen Werte deines Netzes ein.

## Prüfen

`nixos-rebuild build --flake /etc/nixos#<hostname>` muss ohne Fehlermeldung durchlaufen und einen `result`-Symlink im aktuellen Verzeichnis anlegen, der auf einen Pfad unter `/nix/store` zeigt. Nach `switch`: `hostname` gibt `<hostname>` aus, `ip -4 addr show eth0` zeigt `<ip>` mit der gesetzten Präfixlänge statt einer DHCP-Adresse. Da noch kein SSH-Zugang eingerichtet ist (Schritt 3), läuft die Prüfung weiterhin über `pct enter <vmid>`.

## Wenn's schiefgeht

**`error: experimental Nix feature 'flakes' is disabled`** beim ersten `nixos-rebuild build`: Der `echo`-Schritt in `/etc/nix/nix.conf` fehlt oder eine neue Shell wurde nicht geöffnet. Fix: Zeile prüfen, neu einloggen.

**`You must set the option 'boot.loader.grub.device' ...`**: Zeigt, dass `proxmox-lxc.nix` *nicht* importiert wurde – ohne dessen `boot.isContainer = true` hält NixOS die Maschine für Bare-Metal und verlangt einen Bootloader. Fix: `imports`-Zeile prüfen.

**Nach `switch` keine Verbindung mehr, `pct enter` funktioniert aber:** Gateway oder Präfixlänge falsch geraten. Fix: per `pct enter` einloggen, Werte in `configuration.nix` korrigieren, erneut pushen und `switch` wiederholen.

## Rückweg

Vor dem ersten `switch` genügt es, die gepushten Dateien zu löschen (`pct exec <vmid> -- rm -rf /etc/nixos/*`) – am laufenden, weiterhin per DHCP konfigurierten System hat sich nichts geändert. Nach einem `switch` bringt `nixos-rebuild switch --rollback` (siehe Kapitel 7) die vorige, DHCP-basierte Generation zurück; sie existiert, weil das Bootstrap-Template aus Schritt 1 selbst schon eine Generation war.

## Querverweis

Flakes, `flake.nix`/`flake.lock`, `nixosSystem` und Reproduzierbarkeit: Teil I, Kapitel 8 ("Reproduzierbarkeit"). `build` vs. `switch` und Rollback-Mechanik: Kapitel 7. Statische IP-Optionen (`networking.interfaces`, `defaultGateway`) und `imports`: Kapitel 6 bzw. 5.

---

<sup>1</sup> `pct push <vmid> <lokale-datei> <ziel-im-container>` kopiert genau eine Datei vom Proxmox-Host in den Container, unabhängig vom Netzwerkzustand des Gasts; `pct pull` ist das Gegenstück. Quelle: [Proxmox-Community-Zusammenfassung der `pct`-Subcommands](https://gist.github.com/tinoji/7e066d61a84d98374b08d2414d9524f2) – die offizielle Proxmox-Dokumentation war aus dieser Umgebung heraus per Netzzugriff nicht erreichbar, die Syntax ist aber über mehrere unabhängige Community-Quellen deckungsgleich bestätigt.
