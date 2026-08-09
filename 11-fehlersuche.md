---
title: "Fehlersuche"
weight: 11
---

# Fehlersuche

## Lernziele

- Du kannst Nix-Fehlermeldungen strukturiert lesen und setzt `--show-trace` gezielt ein.
- Du nutzt `nix repl` und `nix eval` zum interaktiven bzw. gezielten Auswerten, ohne vollen Rebuild.
- Du durchsuchst `journalctl` gezielt nach Boot- und Dienstfehlern.
- Du erkennst das Muster hinter "infinite recursion"-Fehlern und diagnostizierst Options-Konflikte, statt sie nur mit `mkForce` zuzukleistern.
- Du untersuchst fehlgeschlagene Builds mit dem vollen Log und `--keep-failed`.

## Warum das wichtig ist

Fast jeder Fehler, der dir in den Kapiteln 5 bis 10 begegnet ist, folgt einem von wenigen wiederkehrenden Mustern. Dieses Kapitel löst die Versprechen aus früheren Kapiteln ein – die volle Einführung in `nix repl` und `journalctl`, die dort bewusst verschoben wurden – und bündelt sie zu einer Methode, mit der du Fehler systematisch statt zufällig findest.

## Nix-Fehlermeldungen lesen lernen

Zwei grundverschiedene Fehlerarten, die unterschiedliche Werkzeuge brauchen:

- **Evaluation-Fehler** passieren, *bevor* überhaupt etwas gebaut wird: Syntaxfehler, fehlende Optionen, Typkonflikte, infinite recursion. Werkzeug: `--show-trace`.
- **Build-Fehler** passieren *während* des eigentlichen Bauens: fehlende Header, fehlgeschlagene Tests, Netzwerk-Timeouts. Werkzeug: `nix log` (siehe unten).

Nix' Stack-Traces sind lang, aber nach einem festen Muster aufgebaut: eine Kette von `… while evaluating …`-Zeilen, die von der eigentlichen Fehlerursache bis zu deinem Konfigurationsfile hochreicht. Praktischer Lesetipp: Fang bei der *letzten* Zeile an, die auf eine Datei aus deiner eigenen Konfiguration zeigt (nicht auf Nixpkgs-internes) – das ist fast immer der beste Startpunkt, nicht die oberste Zeile der Meldung.

## `--show-trace` sinnvoll einsetzen

Ohne `--show-trace` kürzt Nix lange Traces mit einem Hinweis wie

```
(stack trace truncated; use '--show-trace' to show the full trace)
```

Diesen Hinweis zu ignorieren und stattdessen zu raten, ist der häufigste Anfängerfehler bei der Fehlersuche überhaupt. Einfach anhängen:

```console
$ nixos-rebuild build --show-trace
```

## `nix repl` zum interaktiven Erkunden

Aus Kapitel 3 kennst du `nix-instantiate --eval` für einzelne Ausdrücke. `nix repl` geht weiter: eine interaktive Umgebung, in der du Werte Schritt für Schritt aufbauen und inspizieren kannst.

```console
$ nix repl --extra-experimental-features flakes nixpkgs
Loading Installable 'flake:nixpkgs#'...
Added 5 variables.
nix-repl> legacyPackages.x86_64-linux.hello.name
"hello-2.12.1"
nix-repl> :q
```

Für eine einzelne Datei:

```console
$ nix repl --file configuration.nix
```

Nützliche Befehle innerhalb der REPL: `:a <expr>` bindet die Attribute eines Sets direkt in den Sichtbereich, `:b <expr>` baut eine Derivation, `:e <expr>` öffnet das Paket bzw. die Funktion im Editor, `:?` zeigt die volle Befehlsliste.

## `nix eval` für gezielte Einzelauswertungen

Willst du nur wissen, welchen *tatsächlichen* Wert eine einzelne Option nach dem Zusammenführen aller Module hat, ohne einen vollen Rebuild zu starten:

```console
$ nix eval .#nixosConfigurations.buch-vm.config.services.openssh.enable
```

Das ist oft schneller als `nixos-option` (Kapitel 5) und funktioniert direkt gegen dein Flake, ohne dass die Konfiguration bereits aktiv sein muss.

## `journalctl` für Boot- und Dienstfehler

Drei Aufrufe, die 90 % der Fälle abdecken:

```console
$ journalctl -u <dienstname>       # Log eines einzelnen Diensts
$ journalctl -b                    # Alles seit dem aktuellen Boot
$ journalctl -b -1                 # Alles seit dem VORHERIGEN Boot – Gold wert nach einem Absturz
$ journalctl -b -p err             # Nur Einträge mit Priorität "error" oder höher
```

`journalctl -b -1` ist besonders wichtig, wenn ein Reboot selbst das Problem war: Nach einem Absturz zeigt `-b` (aktueller Boot) nur den Neustart, nicht die Ursache – die steht im *vorherigen* Boot-Log.

## Infinite Recursion: Ursachen & Muster

Ein minimales, aber typisches Beispiel für einen selbstreferenzierenden Fehler:

```nix
{ config, lib, ... }:
{
  options.services.beispiel.enable = lib.mkOption {
    type = lib.types.bool;
    default = config.services.beispiel.enable;   # Zirkelbezug!
  };
}
```

Der Default verweist auf genau die Option, die er gerade definiert – um den Wert zu berechnen, müsste Nix den Wert schon kennen. Die Fehlermeldung:

```
error: infinite recursion encountered at /etc/nixos/beispiel.nix:4:15:
```

Dieses Muster taucht in der Praxis meist versteckter auf: zwei Optionen, die sich gegenseitig referenzieren, oder ein Default, der über mehrere Modul-Ebenen hinweg indirekt wieder bei sich selbst landet. Der Fix ist immer derselbe: die Zirkularität auflösen – einen festen Wert, einen anderen (nicht-zirkulären) Ausdruck, oder `lib.mkDefault` auf etwas, das *nicht* von der Option selbst abhängt.

## Options-Konflikte diagnostizieren

Die Fehlermeldung aus Kapitel 5 –

```
The unique option `services.httpd.adminAddr' is defined multiple times, in `/etc/nixos/httpd.nix' and `/etc/nixos/configuration.nix'.
```

– nennt bereits beide beteiligten Dateien. Das ist der Diagnose-Startpunkt, nicht `lib.mkForce`. Bevor du erzwingst, lohnt sich die Frage: Warum setzt ein Modul, das ich vielleicht nur wegen einer ganz anderen Funktion importiert habe, überhaupt diese Option? Oft ist die eigentliche Lösung, den unerwünschten Import zu entfernen oder gezielter einzubinden – `mkForce` behebt das Symptom, nicht notwendigerweise die Ursache.

## Fehlgeschlagene Builds: Log lesen, `--keep-failed`

Vollständiges Build-Log eines (auch fehlgeschlagenen) Pakets:

```console
$ nix log .#nixosConfigurations.buch-vm.config.system.build.toplevel
```

Für tiefere Analyse: `--keep-failed` verhindert, dass das temporäre Build-Verzeichnis nach einem Fehlschlag gelöscht wird, und gibt dir den Pfad dazu aus – du kannst dort hineinwechseln und den fehlgeschlagenen Build-Schritt manuell nachvollziehen:

```console
$ nix build --keep-failed .#nixosConfigurations.buch-vm.config.system.build.toplevel
```

## Vollständiges Beispiel

Den Zirkelbezug von oben reproduzieren, den Fehler sehen, reparieren:

```console
$ nixos-rebuild build --show-trace
error: infinite recursion encountered at /etc/nixos/beispiel.nix:4:15:
```

Fix – Default entkoppeln:

```nix
default = false;   # statt config.services.beispiel.enable
```

Danach zur Kontrolle direkt nachschlagen, ohne Rebuild:

```console
$ nix eval .#nixosConfigurations.buch-vm.config.services.beispiel.enable
false
```

> 💡 **Nice to know:** `nix-tree` visualisiert die Abhängigkeiten eines Store-Pfads interaktiv als navigierbaren Baum – deutlich angenehmer als `nix-store -q --requisites` (Kapitel 2) für alles, was über ein paar Zeilen hinausgeht.

> 💡 **Nice to know:** Neben dem "klassischen" (upstream) Nix gibt es inzwischen alternative Implementierungen: **Lix** ist ein community-getragener Fork mit eigenem Installer; **Determinate Nix** ist ein kommerzielles Produkt von Determinate Systems, nach eigener Aussage aber ein *Downstream* mit Patches, die an Upstream zurückfließen, kein Fork. Für dieses Buch spielt das keine Rolle – die hier gezeigten Befehle funktionieren auf allen dreien –, aber der Name kann dir in Foren/Blogposts begegnen, ohne dass sofort klar ist, wovon eigentlich die Rede ist.

## Typische Fehler

**1. Die Trace-Kürzung ignorieren:**

```
(stack trace truncated; use '--show-trace' to show the full trace)
```

*Ursache:* Ohne `--show-trace` zeigt Nix standardmäßig nur einen verkürzten Trace.
*Fix:* Den Hinweis ernst nehmen und `--show-trace` anhängen, statt an der kurzen Meldung zu rätseln.

**2. Selbstreferenzierender Default:**

```
error: infinite recursion encountered at /etc/nixos/beispiel.nix:4:15:
```

*Ursache:* Eine Option (direkt oder über mehrere Ebenen indirekt) referenziert in ihrem eigenen Default ihren eigenen Wert.
*Fix:* Die Zirkularität auflösen – siehe Abschnitt "Infinite Recursion" oben.

## Übung

1. Baue die zirkuläre Option aus diesem Kapitel in deiner Buch-VM nach, beobachte die Fehlermeldung mit `--show-trace`, und behebe sie.
2. Finde mit `journalctl -b -p err` heraus, ob deine Buch-VM seit dem letzten Boot Einträge mit Priorität "error" oder höher geloggt hat – und falls ja, ordne mindestens einen davon einem Dienst zu.

**Lösungsskizze:**

Zu 1: Folge dem Abschnitt "Vollständiges Beispiel" oben.

Zu 2: Individuell, abhängig vom Zustand deiner VM; `journalctl -u <dienst>` liefert danach den vollen Kontext zum jeweiligen Eintrag.

## Zusammenfassung

- Evaluation-Fehler brauchen `--show-trace`, Build-Fehler brauchen `nix log` – unterschiedliche Werkzeuge für unterschiedliche Fehlerarten.
- Stack-Traces liest man am besten von der letzten Zeile in eigenem Code aus, nicht von oben nach unten.
- `nix repl` erlaubt interaktives Erkunden von Werten; `nix eval` liefert gezielt einzelne, bereits zusammengeführte Optionswerte ohne vollen Rebuild.
- `journalctl -b -1` zeigt das Log des *vorherigen* Boots – entscheidend nach einem Absturz.
- "infinite recursion" bedeutet fast immer: eine Option hängt (direkt oder indirekt) von sich selbst ab.
- Options-Konflikte nennen ihre Fundorte bereits in der Fehlermeldung – das ist der Diagnose-Startpunkt, `mkForce` nur die letzte Notlösung.
- `--keep-failed` lässt dich das Verzeichnis eines fehlgeschlagenen Builds manuell untersuchen.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `--show-trace`, Eval- vs. Build-Fehler-Unterscheidung, Lesetechnik | [iampavel.dev – How to Actually Read Nix Error Messages](https://iampavel.dev/blog/how-to-read-nix-error) |
| `nix repl` (Syntax, `--file`, `--expr`, `--extra-experimental-features`) | [Nix Reference Manual – nix repl](https://nix.dev/manual/nix/2.30/command-ref/new-cli/nix3-repl.html) |
| REPL-Befehle `:a`/`:b`/`:e`/`:q`/`:?` | [NixOS & Flakes Book – Debugging](https://nixos-and-flakes.thiscute.world/best-practices/debugging) |
| `journalctl -u`/`-b`/`-b -1`/`-p err` | Standard-systemd-Dokumentation (allgemeines Linux-Wissen) |
| `error: infinite recursion encountered at …` | Wiederkehrendes, gut dokumentiertes Muster, u. a. [NixOS-Discourse](https://discourse.nixos.org/t/infinite-recursion-encountered-by-making-module-configurable/23508) |
| "unique option … defined multiple times" | NixOS Manual – Modularity (siehe Kapitel 5) |
| `nix log`, `--keep-failed` | Nix Reference Manual (Command-Line-Referenz) |
| `nix-tree` | Community-Tool, siehe Projekt-Repository |
| Lix, Determinate Nix (Einordnung) | [Lix Systems](https://git.lix.systems/lix-project/lix-installer), [Determinate Systems – Blog](https://determinate.systems/blog/installer-dropping-upstream/) |
