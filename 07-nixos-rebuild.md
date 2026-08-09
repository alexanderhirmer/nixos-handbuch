---
title: "nixos-rebuild im Griff"
weight: 7
---

# nixos-rebuild im Griff

## Lernziele

- Du kennst die wichtigsten `nixos-rebuild`-Subcommands und weißt, wann welcher passt.
- Du kannst Generationen einsehen und gezielt zwischen ihnen wechseln.
- Du kannst über Bootmenü und CLI zurückrollen.
- Du verstehst den Unterschied zwischen einem Build-Fehler und einem Aktivierungsfehler.

## Warum das wichtig ist

`nixos-rebuild` ist der Befehl, mit dem aus der Theorie in Kapitel 2 – Generationen, atomare Aktivierung – gelebte Praxis wird. Wer riskante Änderungen erst mit `test` statt direkt mit `switch` ausprobiert, braucht die meisten der hier beschriebenen Rollback-Wege im Ernstfall gar nicht erst.

## Die Subcommands im Überblick

| Subcommand | Baut | Aktiviert jetzt | Wird Boot-Default |
|---|---|---|---|
| `switch` | ja | ja | ja |
| `boot` | ja | nein (erst beim nächsten Neustart) | ja |
| `test` | ja | ja | **nein** |
| `build` | ja | nein | nein – legt nur `./result` an |
| `dry-build` | zeigt nur, was gebaut würde | nein | nein |
| `dry-activate` | ja | nein – zeigt nur, was sich bei Aktivierung ändern würde | nein |
| `build-vm` | ja | – startet stattdessen eine QEMU-VM mit der Konfiguration | nein |
| `build-vm-with-bootloader` | wie `build-vm`, testet zusätzlich den echten Bootloader | – | nein |

(Quelle: NixOS-Wiki und `nixos-rebuild`-Man-Page.)

Für riskante Änderungen ist `test` der pragmatische Standardweg: Du siehst sofort, ob etwas kaputtgeht, und ein einfacher Reboot bringt dich zur alten, weiterhin als Boot-Default eingetragenen Generation zurück – ganz ohne Bootmenü-Navigation. `dry-activate` geht noch vorsichtiger: Du bekommst angezeigt, was sich ändern würde, ohne dass irgendetwas am laufenden System angefasst wird.

## Generationen verwalten

```console
$ nixos-rebuild list-generations
Generation  Build-date            NixOS version         Kernel   Configuration Revision  Specialisation
  95         2026-08-08 09:12:03  26.05.XXXX.abcdef123   6.18     …
  94         2026-08-07 21:03:41  26.05.XXXX.abcdef123   6.18     …
  93 current 2026-08-07 18:47:16  26.05.XXXX.abcdef123   6.18     …
```

(Format sinngemäß aus der NixOS-Wiki; deine tatsächlichen Zeilen sehen natürlich anders aus.) Das ist dieselbe Kette von `system-<N>-link`-Symlinks aus Kapitel 2, nur lesbar aufbereitet.

## Rollback: Bootmenü & CLI

Am Bootmenü selbst: Einfach die gewünschte ältere Generation auswählen, direkt bootbar, keine Kommandozeile nötig.

Von der laufenden Kommandozeile aus:

```console
# nixos-rebuild switch --rollback
```

Das baut *nichts* neu, sondern aktiviert die Generation, die vor der aktuellen im Systemprofil `/nix/var/nix/profiles/system` stand. `--rollback` lässt sich auch mit `boot` oder `test` kombinieren, wenn du den alten Zustand erst testen statt gleich zum Boot-Default machen willst.

> ⚠️ Ungeprüft/versionsabhängig: Ob `nixos-rebuild` inzwischen eingebaut mehr als eine Generation auf einmal zurückspringen kann, konnte ich beim Schreiben nicht abschließend verifizieren – Stand einer community-dokumentierten Quelle war das (jedenfalls bis Ende 2024) nicht eingebaut vorgesehen. Der zuverlässige Fallback, der schon länger funktioniert: die gewünschte Generation direkt ansteuern –
> ```console
> # nix-env --profile /nix/var/nix/profiles/system --switch-generation 93
> # /nix/var/nix/profiles/system/bin/switch-to-configuration switch
> ```
> `list-generations` von oben liefert dir die Nummer.

## Was bei einem Fehlschlag technisch passiert

Zwei ganz unterschiedliche Fehlerarten:

**Build-Fehler** – ein Syntaxfehler, eine fehlende Option, ein Paket, das nicht baut: Der komplette Vorgang bricht *vor* jeder Änderung am System ab. Das alte System läuft unverändert weiter, keine neue Generation wird registriert. Das ist der ungefährliche Fall.

**Aktivierungsfehler** – der Build war erfolgreich, aber beim Umschalten geht etwas schief, z. B. weil ein systemd-Dienst nicht sauber neu startet. In diesem Fall *ist* die neue Generation bereits registriert, aber das laufende System kann in einem Mischzustand hängen bleiben. `nixos-rebuild` meldet das typischerweise mit einer Warnung nach dem Muster `warning: error(s) occurred while switching to the new configuration`. Genau dafür ist `test` (siehe Tabelle oben) die sicherere Wahl bei unsicheren Änderungen: Schlägt die Aktivierung fehl, bringt dich ein Reboot zur alten, weiterhin als Default eingetragenen Generation zurück, ohne dass du überhaupt zum Bootmenü musst.

## Vollständiges Beispiel

Ein typischer, vorsichtiger Ablauf für eine Änderung, der du nicht zu 100 % traust:

```console
# nixos-rebuild test
# systemctl status <betroffener-dienst>
# curl localhost:<port>   # oder was auch immer die Änderung prüft
```

Läuft alles wie erwartet:

```console
# nixos-rebuild switch
```

Läuft es *nicht* wie erwartet, reicht ein Reboot – `test` hat den Boot-Default ja nie verändert.

Zum Vergleich: Denselben Ablauf einmal komplett zurückrollen, weil sich im Nachhinein doch ein Problem zeigt:

```console
# nixos-rebuild list-generations
# nixos-rebuild switch --rollback
```

> 💡 **Nice to know:** `--target-host` (und `--build-host`) erlauben, ein entferntes System zu aktualisieren, ohne dich vorher einzuloggen – der eigentliche Rebuild läuft dann remote. Das ist die Grundlage für den Mehrere-Maschinen-Workflow in Kapitel 14.

> 💡 **Nice to know:** Mit `specialisation` lassen sich alternative Boot-Konfigurationen definieren, die von deiner Hauptkonfiguration abweichen – z. B. ein Profil mit anderer Netzwerk-/Proxy-Konfiguration, das im Bootmenü als zusätzliche Option auftaucht, statt eine komplette Zweitkonfiguration zu pflegen. Für den Einstieg reicht dieses Kapitel; taucht dir der Begriff später wieder unter, weißt du jetzt, wonach du suchst.

## Typische Fehler

**1. `--flake` auf einem Flake-basierten System vergessen:**

```
error: getting status of '/etc/nixos/configuration.nix': No such file or directory
```

*Ursache:* Ohne `--flake` sucht `nixos-rebuild` nach der klassischen `configuration.nix` – auf einem reinen Flake-System (Kapitel 8) existiert die aber gar nicht.
*Fix:* `--flake '/etc/nixos#hostname'` (oder den passenden Pfad) mit angeben.

**2. Aktivierung schlägt teilweise fehl:**

```
warning: error(s) occurred while switching to the new configuration
```

*Ursache:* Der Build war erfolgreich, aber mindestens ein Dienst konnte beim Umschalten nicht sauber neu gestartet werden.
*Fix:* Betroffenen Dienst mit `systemctl status <name>` prüfen (volle Fehlersuche folgt in Kapitel 11); im Zweifel per Reboot zur alten Generation zurück, wenn `test` statt `switch` verwendet wurde.

## Übung

1. Ändere etwas Kleines in deiner Buch-VM (z. B. `networking.hostName`), aktiviere es zunächst mit `nixos-rebuild test`, prüfe die Wirkung – und mach dir bewusst, dass ein Reboot jetzt zur alten Konfiguration zurückkehren würde. Erst danach mit `switch` endgültig machen.
2. Liste deine Generationen mit `list-generations`, nimm eine weitere kleine Änderung vor, und roll sie anschließend mit `--rollback` zurück. Prüfe mit `list-generations`, dass die aktuelle Generation wieder die alte ist.

**Lösungsskizze:**

Zu 1 und 2: Folge den Befehlen im Abschnitt "Vollständiges Beispiel" oben.

## Zusammenfassung

- `switch`, `boot`, `test`, `build`, `dry-build`, `dry-activate`, `build-vm` und `build-vm-with-bootloader` unterscheiden sich darin, ob gebaut, aktiviert und/oder zum Boot-Default gemacht wird.
- `test` ist der pragmatische Weg für riskante Änderungen: aktiviert, aber kein neuer Boot-Default, ein Reboot reicht als Fallback.
- `list-generations` zeigt dieselbe Generationenkette aus Kapitel 2, nur lesbar.
- `nixos-rebuild switch --rollback` aktiviert die vorherige Generation, ohne neu zu bauen; mehr als einen Schritt zurück braucht den `switch-to-configuration`-Fallback.
- Build-Fehler sind ungefährlich (nichts ändert sich); Aktivierungsfehler können einen Mischzustand hinterlassen – genau dafür ist `test` da.
- `--target-host` (Kapitel 14) und `specialisation` sind zwei Wege, die über den Alltag dieses Kapitels hinausgehen.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `switch`/`boot`/`test`/`build`/`dry-build`/`dry-activate`/`build-vm`/`build-vm-with-bootloader` | [NixOS-Wiki – nixos-rebuild](https://wiki.nixos.org/wiki/Nixos-rebuild), [Man-Page (mankier)](https://www.mankier.com/8/nixos-rebuild) |
| `nixos-rebuild list-generations` (Ausgabeformat) | [NixOS-Wiki – nixos-rebuild](https://wiki.nixos.org/wiki/Nixos-rebuild) |
| `nixos-rebuild switch --rollback` | [Man-Page (mankier)](https://www.mankier.com/8/nixos-rebuild), Praxisbeispiel (chenlijun99/dotfiles) |
| `nix-env --profile /nix/var/nix/profiles/system --switch-generation`, `switch-to-configuration switch` | [NixOS Manual](https://nixos.org/manual/nixos/stable/), [NixOS-Wiki – nixos-rebuild](https://wiki.nixos.org/wiki/Nixos-rebuild) |
| `--target-host`, `--build-host` | [Man-Page (mankier)](https://www.mankier.com/8/nixos-rebuild) |
| `specialisation` | NixOS Manual (Erwähnung im Kontext Proxy-Konfiguration und Image-Varianten) |
