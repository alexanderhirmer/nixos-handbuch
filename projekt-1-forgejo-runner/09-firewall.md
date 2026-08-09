---
title: "Firewall"
weight: 9
---

# Schritt 9: Die Firewall lässt nur Port 40 herein

## Ziel

Die Firewall lässt ausschließlich Port `<ssh-port>` eingehend herein; alles, was der Runner sonst braucht (LDAP, Forgejo), läuft ausgehend und ist von dieser Filterung gar nicht betroffen.

## Voraussetzung

Schritte 1–8 sind abgeschlossen (Baseline inkl. `sssd`). Die NixOS-Firewall läuft bisher auf Standardeinstellung – aktiv, aber ohne projekteigene Freigaben außer dem, was `ssh.nix` (Schritt 5) über `services.openssh.openFirewall`/`allowedTCPPorts` ggf. schon geöffnet hat.

## Durchführung

**1. `firewall.nix` anlegen** (Inhalt siehe unten), in `default.nix` importieren, testweise bauen, dann erst aktivieren:

```console
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

**2. Von einem anderen Rechner aus sofort gegenprüfen**, dass SSH noch erreichbar ist, *bevor* die aktuelle Sitzung geschlossen wird:

```console
$ ssh -p <ssh-port> <admin-user>@<ip>
```

**3. Optional, als zusätzliche Schicht auf dem Proxmox-Host**: Damit ein Fehler in der Gast-Firewall nicht die einzige Verteidigungslinie ist, kann Proxmox selbst zusätzlich filtern (siehe Warnbox unten für den genauen Mechanismus).

## Dateien

```nix
# <repo-root>/modules/baseline/firewall.nix
{ ... }:
{
  networking.firewall = {
    enable = true;               # ist ohnehin die Voreinstellung
    allowedTCPPorts = [ <ssh-port> ];
    allowedUDPPorts = [ ];       # nichts – der Runner braucht nur ausgehende Verbindungen
    allowPing = false;
  };
}
```

```diff
 # <repo-root>/modules/baseline/default.nix
 { ... }:
 {
   imports = [
     ./users.nix
     ./sudo.nix
     ./ssh.nix
     ./fail2ban.nix
     ./ldap.nix
+    ./firewall.nix
   ];
 }
```

> 💡 **Nice to know – der eigentliche Lehrpunkt:** `networking.firewall` filtert nur *eingehende* Verbindungen; für selbst aufgebaute ausgehende gibt es in `nixos/modules/services/networking/firewall.nix` keine Default-Restriktion (kein „deny outbound"). Der Forgejo-Runner meldet sich per Long-Polling *aktiv* bei `<forgejo-url>` – eine ausgehende Verbindung, braucht also **keinen** offenen Port. Genauso die LDAP-Anbindung an `<ldap-uri>` aus Schritt 8. Eingehend braucht dieser Host nur SSH. Quelle: [nixpkgs, `services/networking/firewall.nix`](https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/networking/firewall.nix).

> 💡 **Nice to know:** `networking.nftables.enable` (Default `false`) ändert weniger als der Name suggeriert. Nixpkgs baut `pkgs.iptables` standardmäßig mit `nftablesCompat = true`: `iptables`/`ip6tables` sind Symlinks auf `xtables-nft-multi` und übersetzen Regeln ohnehin in den nf_tables-Kernel-Mechanismus. „iptables" oder „nftables" als Backend ist im Kern dieselbe Kernel-Infrastruktur. Quelle: [nixpkgs, `pkgs/by-name/ip/iptables/package.nix`](https://github.com/NixOS/nixpkgs/blob/release-26.05/pkgs/by-name/ip/iptables/package.nix).

> ⚠️ Ungeprüft: Ob die NixOS-Firewall in einem *unprivilegierten* Proxmox-LXC-Container zuverlässig funktioniert, ließ sich in meiner Recherche-Umgebung nicht an einer Proxmox-Primärquelle verifizieren (`pve.proxmox.com` war nicht erreichbar). Mehrere übereinstimmende Community-Quellen (Proxmox-Forum, LXC-Projekt) sagen: nftables/iptables funktionieren in einem unprivilegierten Container grundsätzlich, sofern `CAP_NET_ADMIN` im eigenen Namespace vorhanden ist – das ist bei Standard-Proxmox-Containern der Fall, auch ohne das für Docker-artiges Nesting nötige `features: nesting=1`. Trotzdem: nach `switch` per `nft list ruleset` im Container prüfen, ob überhaupt eine Regelmenge geladen ist.

> ⚠️ Zweite, unabhängige Filterschicht auf dem Host: Proxmox hat eine eigene, vom Gast unabhängige Firewall – pro NIC über `firewall=1` im `net0`-Parameter aktiviert (`pct set <vmid> --net0 name=eth0,bridge=vmbr0,firewall=1,...`), Regeln über `[OPTIONS]`/`[RULES]` in `/etc/pve/firewall/<vmid>.fw` (`enable: 1` schaltet scharf). Auch das nur gegen Sekundärquellen abgeglichen, nicht gegen die Proxmox-Doku selbst – vor Produktiveinsatz gegenprüfen. Für dieses Projekt optional, die NixOS-Firewall allein reicht.

## Prüfen

- Von einem zweiten Rechner: `ssh -p <ssh-port> <admin-user>@<ip>` gelingt weiterhin.
- Ein Verbindungsversuch auf einen beliebigen anderen Port (z. B. `nc -zv <ip> 22` oder `nc -zv <ip> 8080`) schlägt fehl bzw. hängt (kein Reset, kein Connect).
- Im Container zeigt `journalctl -u sssd --since -5m`, dass sssd trotz aktiver Firewall weiterhin erfolgreich mit `<ldap-uri>` kommuniziert (ausgehend also unbeeinträchtigt).

## Wenn's schiefgeht

**SSH-Verbindung bricht nach `switch` ab und baut sich nicht neu auf:** Meist ein Tippfehler beim Port oder ein Widerspruch zwischen `ssh.nix` und `firewall.nix`. Rettungsanker: `pct enter <vmid>` auf dem Proxmox-Host öffnet eine Root-Shell **ohne** Netzwerk, unabhängig von jeder Gast-Firewall-Regel; von dort `firewall.nix` korrigieren und neu bauen.

**LDAP-Login (Schritt 8) funktioniert plötzlich nicht mehr:** Deutet auf eine restriktivere Proxmox-Host-Firewall hin, nicht auf `firewall.nix` selbst – NixOS blockt standardmäßig keinen ausgehenden Verkehr. Mit `pct enter` prüfen, ob `ldapsearch -x -H <ldap-uri> ...` vom Container aus überhaupt noch antwortet.

**`nixos-rebuild build` meldet keinen Fehler, aber nach `switch` ist der Container komplett unerreichbar:** Falls zusätzlich eine restriktive Proxmox-Host-Firewall aktiv ist (siehe Warnbox), blockt sie unabhängig von der Gast-Konfiguration. Über `pct enter` einsteigen und `/etc/pve/firewall/<vmid>.fw` auf dem *Host* prüfen.

## Rückweg

`./firewall.nix` aus `imports` in `default.nix` entfernen und rebuilden – die Firewall fällt auf reinen NixOS-Default zurück (aktiv, aber ohne projekteigene Freigaben; SSH bleibt nur offen, falls `ssh.nix` selbst `openFirewall` setzt). Bei komplettem Aussperren: `pct enter <vmid>`, Datei manuell korrigieren, `nixos-rebuild switch` von innen.

## Querverweis

Kapitel 6, „Netzwerk-Grundlagen & Firewall" ([06-alltagsbetrieb.md](../06-alltagsbetrieb.md)): Dort werden `networking.firewall.enable`, `allowedTCPPorts` und `allowPing` erstmals eingeführt, inklusive des Hinweises, dass `services.openssh.openFirewall` denselben Mechanismus nur bequemer macht – hier wird er stattdessen bewusst explizit in einer eigenen Datei gepflegt.
