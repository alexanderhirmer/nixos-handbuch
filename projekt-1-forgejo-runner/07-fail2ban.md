---
title: "fail2ban"
weight: 7
---

# Schritt 7: `fail2ban` ist aktiv und schützt `sshd` auf Port 40

## Ziel

`fail2ban` läuft, überwacht das `sshd`-Jail automatisch auf dem in Schritt 5 gesetzten Port `<ssh-port>` und sperrt IPs nach wiederholten Fehlversuchen zeitlich befristet.

## Voraussetzung

Schritt 5 ist abgeschlossen (`services.openssh.ports = [ 40 ];` steht bereits in `modules/baseline/ssh.nix`, die Firewall ist über `openFirewall` aktiv).

## Durchführung

**1. Modul anlegen** und **2. `default.nix` um eine Zeile erweitern** (Inhalt siehe "Dateien").

**3. Warum kein eigenes `jails.sshd` nötig ist:** NixOS bringt laut Beschreibung der Option `services.fail2ban.jails` im Quellcode bereits ein vorkonfiguriertes `sshd`-Jail mit – wörtlich: "NixOS comes with a default sshd jail […] This module sets [LogLevel] to VERBOSE if not set otherwise". Technisch passiert das über zwei Merge-Bausteine in `fail2ban.nix`: Ein `DEFAULT`-Jail setzt u. a. `backend = "systemd"` (fail2ban liest Login-Fehlschläge direkt aus dem systemd-Journal, kein Log-Datei-Parsing nötig) und `ignoreip = "127.0.0.1/8 ::1 …"`; ein zweiter Block, aktiv sobald `services.openssh.enable` wahr ist, setzt `sshd.settings.port = lib.mkDefault (lib.concatMapStringsSep "," toString config.services.openssh.ports);` – die Portliste kommt also automatisch aus Schritt 5, ohne dass dieses Modul sie kennen muss. Passend dazu hebt derselbe Code `services.openssh.settings.LogLevel` per `mkDefault` auf `"VERBOSE"` an, weil das `sshd`-Jail sonst zu wenig im Journal sieht, um Fehlversuche zu erkennen. `modules/baseline/ssh.nix` (Schritt 5) setzt `LogLevel` nicht selbst, der `mkDefault` greift also ungestört.

Die Jail-Syntax selbst hat sich in Nixpkgs historisch geändert: `services.fail2ban.jails.<name>` akzeptiert heute entweder einen rohen `lines`-String (alte, weiterhin unterstützte Form) oder – empfohlen – ein Submodule mit `enabled`/`filter`/`settings`; die früher separate Option `services.fail2ban.extraSettings` wurde entfernt zugunsten von `services.fail2ban.jails.DEFAULT.settings`. Dieses Modul nutzt ausschließlich die aktuelle Submodule-Form, indirekt über die globalen Kurzoptionen `maxretry`/`bantime`/`bantime-increment`, die selbst wieder in `jails.DEFAULT.settings` einfließen.

**4. `ignoreIP` bewusst leer gelassen:** `127.0.0.1/8` und `::1` sind ohnehin immer ausgenommen; ein zusätzliches vertrauenswürdiges Management-Netz, das man eintragen könnte, steht in diesem Projekt in keiner Platzhaltertabelle – wer eines hat, ergänzt `services.fail2ban.ignoreIP = [ "10.20.0.0/24" ];` o. Ä.

**5. Testen, dann anwenden:**

```console
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

> 💡 **Nice to know:** `bantime-increment.enable = true;` lässt fail2ban die Sperrzeit bei wiederholt derselben IP progressiv verlängern (laut Options-Beschreibung standardmäßig Faktor 1, 2, 4, 8, 16 … auf Basis von `bantime`), ohne dass `multipliers` oder eine eigene `formula` nötig wären – die Defaults dafür reichen.

## Dateien

`<repo-root>/modules/baseline/fail2ban.nix` (neu):

```nix
# <repo-root>/modules/baseline/fail2ban.nix
{ ... }:
{
  services.fail2ban = {
    enable = true;
    maxretry = 3;
    bantime = "1h";
    bantime-increment.enable = true;
  };
}
```

`<repo-root>/modules/baseline/default.nix` (geändert, eine Zeile):

```nix
  imports = [
    ./users.nix
    ./sudo.nix
    ./ssh.nix
+   ./fail2ban.nix
  ];
```

## Prüfen

- `systemctl status fail2ban` zeigt `active (running)`.
- `fail2ban-client status` listet `sshd` unter "Jail list".
- `fail2ban-client status sshd` zeigt einen Block mit Filter- (u. a. "Currently failed", "Total failed") und Actions-Zeilen ("Currently banned", "Total banned", "Banned IP list") – nach einem absichtlichen Fehlversuch von einer Testmaschine aus (`ssh -p 40 nichtvorhanden@<ip>` mehrfach mit falschem Key/Passwort) steigt "Currently failed" sichtbar an.

## Wenn's schiefgeht

**Eigene IP nach dem Testen versehentlich gesperrt:** `fail2ban-client status sshd` zeigt sie unter "Banned IP list". Fix: von einer anderen Quelle oder via `pct enter <vmid>` einsteigen und `fail2ban-client set sshd unbanip <ip>` ausführen.

**Warnung "fail2ban can not be used without a firewall" beim Rebuild** (wörtlich aus dem Nixpkgs-Modul): Ursache ist `networking.firewall.enable = false` bzw. `networking.nftables.enable = false` irgendwo in der Konfiguration. Fix: Firewall aktiv lassen (Standard, wird in Schritt 9 ohnehin formalisiert).

**`fail2ban-client status sshd` bleibt dauerhaft bei "Currently failed: 0" trotz absichtlicher Fehlversuche:** Meist, weil `services.openssh.settings.LogLevel` irgendwo explizit gesetzt wurde (normale Priorität schlägt `mkDefault`) und dadurch unter `VERBOSE` liegt. Fix: `LogLevel` nicht selbst setzen oder explizit auf `"VERBOSE"` (oder höher) heben.

## Rückweg

`./fail2ban.nix` aus der `imports`-Liste in `default.nix` entfernen (oder `services.fail2ban.enable = false;` setzen) und erneut `nixos-rebuild switch --flake /etc/nixos#<hostname>` ausführen.

## Querverweis

Teil I, Kapitel 6 ("Alltagsbetrieb") führt das Muster `services.<name>.enable = true;` samt `settings`/Kurzoptionen ein. Kapitel 11 ("Fehlersuche") erklärt `journalctl -u <dienstname>` – hilfreich, um `journalctl -u fail2ban` bei Problemen mit dem Jail direkt einzusehen.
