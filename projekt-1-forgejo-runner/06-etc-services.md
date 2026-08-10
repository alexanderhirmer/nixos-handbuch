---
title: "Exkurs: /etc/services"
weight: 6
---

# Schritt 6: Geklärt ist, ob `/etc/services` nach der Port-Umstellung angepasst werden muss

## Ziel

Belegt ist, dass `/etc/services` unter NixOS für `sshd`, `fail2ban` und die Firewall irrelevant ist – alle drei beziehen Port `<ssh-port>` direkt aus der Nix-Konfiguration, nicht per Namens-Lookup aus dieser Datei. Keine Konfigurationsänderung, nur Recherche und Nachweis.

## Voraussetzung

Schritt 5 ist abgeschlossen; `sshd` läuft bereits auf Port `<ssh-port>`.

## Durchführung

Reine Lesebefehle, nichts wird verändert.

```console
$ readlink -f /etc/services
$ getent services ssh
$ systemctl cat sshd.service
$ grep -i '^Port' /etc/ssh/sshd_config
```

`readlink -f /etc/services` zeigt einen Pfad unter `/nix/store/…-iana-etc-…/etc/services`. Erzeugt wird dieser Symlink im NixOS-Kernmodul `nixos/modules/config/networking.nix` (`services.source = pkgs.iana-etc + "/etc/services";`) – ungated, also immer aktiv, unabhängig davon, ob `services.openssh` überhaupt läuft ([Quelle](https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/config/networking.nix)). `pkgs.iana-etc` baut keine eigene Datenbank, sondern lädt ein fertiges Release-Tarball von `Mic92/iana-etc` (Republishing der IANA-Port-Registry) und kopiert `services`/`protocols` nach `$out/etc` ([Quelle](https://github.com/NixOS/nixpkgs/blob/release-26.05/pkgs/by-name/ia/iana-etc/package.nix)). Schreibbar ist die Datei nicht: Ziel ist ein Nix-Store-Pfad (read-only), und selbst ein erzwungener Austausch würde beim nächsten `switch` überschrieben, weil `/etc` laut `nixos/modules/system/etc/etc.nix` bei jeder Aktivierung komplett aus `environment.etc` neu aufgebaut wird.

`systemctl cat sshd.service` plus der `grep` zeigen: `ExecStart` verweist nur auf `/etc/ssh/sshd_config`, dort steht `Port 40` als Literal. Im Quellcode entsteht das über `lib.map (port: "Port ${toString port}") cfg.ports` – eine Zahl aus `services.openssh.ports`, kein Name. Genauso `fail2ban` (`sshd.settings.port = lib.mkDefault (lib.concatMapStringsSep "," toString config.services.openssh.ports);`) und die Firewall (`networking.firewall.allowedTCPPorts = lib.optionals cfg.openFirewall cfg.ports;`) – beide dieselbe Zahlenliste, keiner ruft `getservbyname`/`getent` auf. `getent services ssh` liefert entsprechend weiterhin den statischen IANA-Standardeintrag für "ssh" (Port 22) – kein Fehler, sondern der Beweis, dass diese Datei von der tatsächlichen Serverkonfiguration nichts weiß.

**Ergebnis:** Die vorläufige Einschätzung aus `ENTSCHEIDUNGEN.md` ist bestätigt – `/etc/services` bleibt unangetastet.

> 💡 **Nice to know:** Bräuchte ein Legacy-Tool doch einen eigenen Eintrag: keine "Zusatzeintrag"-Option vorhanden, nur die ganze Datei ersetzen – `environment.etc."services".source = lib.mkForce (…);` (`mkForce` nötig, sonst der aus Kapitel 5 bekannte Konflikt "defined multiple times").

## Dateien

Keine – reiner Prüfschritt ohne Konfigurationsänderung.

## Prüfen

`readlink -f /etc/services` zeigt einen `/nix/store/…`-Pfad; `grep -i '^Port' /etc/ssh/sshd_config` zeigt `Port 40`; `getent services ssh` zeigt weiterhin Port 22 – zusammen der Beleg, dass beides unabhängig voneinander existiert.

## Wenn's schiefgeht

**Versuch, `/etc/services` direkt zu editieren, scheitert mit "Permission denied":** Erwartetes Verhalten, kein Fehler – Ziel des Symlinks liegt im read-only Nix Store. Fix: nicht nötig, siehe oben; Port-Änderungen laufen ausschließlich über `modules/baseline/ssh.nix` (Schritt 5).

## Rückweg

Entfällt – dieser Schritt hat nichts verändert.

## Querverweis

Teil I, Kapitel 2 ("Das Nix-Modell") erklärt, warum der Nix Store read-only ist; Kapitel 5 ("Das Modulsystem") erklärt den Konfliktfehler "defined multiple times" und `lib.mkForce`.
