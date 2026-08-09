---
title: "Das Modulsystem"
weight: 5
---

# Das Modulsystem

## Lernziele

- Du kannst `options` und `config` als die zwei Seiten eines Moduls auseinanderhalten.
- Du verstehst, wie `imports` mehrere Module zu einer Konfiguration zusammenführt.
- Du kannst `mkIf`, `mkDefault`, `mkForce` und `mkMerge` gezielt einsetzen, um Prioritätskonflikte zu lösen.
- Du hast einen groben Überblick, was `lib` an Werkzeugen mitbringt.
- Du kannst ein eigenes kleines Modul mit Enable-Flag schreiben.

## Warum das wichtig ist

Ab jetzt ist eine NixOS-Konfiguration kein einzelnes Nix-File mehr, sondern ein Baum aus Modulen, die sich ergänzen, überschreiben oder in Konflikt geraten. Ohne dieses Modell wirken Merge-Fehler oder die Frage "warum gewinnt mein Wert nicht" willkürlich. Mit ihm sind sie mechanisch nachvollziehbar.

## `options` vs. `config` – die zwei Seiten eines Moduls

Ein Modul kann zwei Dinge zurückgeben:

- **`options`** – WELCHE Einstellungen es überhaupt gibt: Name, Typ, Default, Beschreibung. Das ist die *Deklaration*.
- **`config`** – WELCHE Werte für (eigene oder fremde) Optionen tatsächlich gesetzt werden. Das ist die *Definition*.

Fast jede `configuration.nix`, die du im Netz siehst, besteht nur aus `config` – meist sogar ohne das Wort hinzuschreiben, weil es implizit ist. Du definierst Werte für Optionen, die anderswo (in Nixpkgs' eigenen Modulen) bereits deklariert wurden. Erst ein eigenes Modul mit einer *neuen* Option braucht auch `options`.

Die volle Struktur sieht so aus:

```nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.meinDienst;
in
{
  options.services.meinDienst = {
    enable = mkEnableOption "meinen Dienst";
  };

  config = mkIf cfg.enable {
    environment.systemPackages = [ pkgs.hello ];
  };
}
```

Das `let cfg = config.services.meinDienst;` ist reine Konvention, keine Pflicht – aber eine, die dir in praktisch jedem Nixpkgs-Modul begegnet, weil sie spätere Tipp- und Pfadfehler vermeidet.

## `imports` – Module komponieren

`imports` (die modulsystem-eigene Liste, nicht zu verwechseln mit dem Sprachfeature `import` aus Kapitel 3) bindet weitere Module ein. Jedes eingebundene Modul kann selbst wieder `options` und/oder `config` mitbringen, und alle werden zu einer einzigen, zusammengeführten Konfiguration verrechnet.

Für Optionen, die eine Liste erwarten (z. B. `environment.systemPackages`), werden mehrere Definitionen aus verschiedenen Modulen einfach aneinandergehängt – die Definition in `configuration.nix` selbst landet dabei am Ende der zusammengeführten Liste. Für Optionen, die nur *einen* Wert sinnvoll haben können (z. B. einen einzelnen String), führt das bei zwei widersprüchlichen Definitionen zu einem Konfliktfehler (siehe "Typische Fehler" unten) – Nix kann hier nicht raten, welcher Wert gemeint ist.

## `mkIf`, `mkDefault`, `mkForce`, `mkMerge` – Prioritäten verstehen

- **`mkIf cond value`** – `value` gilt nur, wenn `cond` wahr ist; sonst so, als wäre nichts definiert worden. Das Standardmuster `config = mkIf cfg.enable { ... };` sorgt dafür, dass ein Modul im deaktivierten Zustand wirklich nichts tut, ganz ohne verschachteltes `if/then/else`.
- **`mkDefault value`** – setzt `value` mit niedrigerer Priorität als eine gewöhnliche Definition. Jede "normale" Zuweisung an derselben Option gewinnt automatisch gegen `mkDefault`, ohne Konflikt. Viele Nixpkgs-Module nutzen das für sinnvolle Voreinstellungen, die du unbemerkt überschreiben kannst.
- **`mkForce value`** – das Gegenteil: höchste Priorität, gewinnt gegen praktisch jede andere Definition. Genau damit löst man den Konfliktfehler aus dem vorigen Abschnitt: `services.httpd.adminAddr = lib.mkForce "bob@example.org";` erzwingt diesen einen Wert, egal was anderswo steht.
- **`mkMerge [ a b c ]`** – führt mehrere `config`-Blöcke zu einem zusammen. Praktisch, um innerhalb eines Moduls mehrere bedingte Abschnitte sauber getrennt zu halten, statt alles in ein verschachteltes `if/then/else` zu quetschen.
- **`mkBefore value`** – ein Spezialfall für listenwertige Optionen: sorgt dafür, dass dieser Eintrag *vor* anderen in der zusammengeführten Liste steht, z. B. `boot.kernelModules = mkBefore [ "kvm-intel" ];`, damit dieses Modul garantiert vor anderen geladen wird.

## `lib` – ein kurzer Überblick

`lib` ist Nixpkgs' Standardbibliothek – deutlich mehr, als in ein Kapitel passt. Für den Einstieg reicht diese grobe Einteilung:

- **Modul-Mechanik**: `mkIf`, `mkDefault`, `mkForce`, `mkMerge`, `mkOption`, `mkEnableOption` – das, was du in diesem Kapitel gerade gelernt hast.
- **Datenverarbeitung**: Funktionen wie `mapAttrs`, `filter`, `optional`, `optionals` für Listen und Attribute Sets.
- **`lib.types`**: das Typsystem für `mkOption` (`str`, `int`, `bool`, `listOf`, `attrsOf`, `enum`, …) – dazu mehr im Nice-to-know unten.

## Ein eigenes kleines Modul schreiben

## Vollständiges Beispiel

Ein Modul, das – wenn aktiviert – einen frei konfigurierbaren Begrüßungstext nach `/etc/motd` schreibt:

```nix
# modules/begruessung.nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.begruessung;
in
{
  options.services.begruessung = {
    enable = mkEnableOption "eine Begrüßungsnachricht beim Login";

    text = mkOption {
      type = types.str;
      default = "Willkommen auf ${config.networking.hostName}!";
      description = "Der Text, der beim Login als /etc/motd angezeigt wird.";
    };
  };

  config = mkIf cfg.enable {
    environment.etc."motd".text = cfg.text;
  };
}
```

Eingebunden in `configuration.nix`:

```nix
{
  imports = [ ./modules/begruessung.nix ];

  services.begruessung.enable = true;
  services.begruessung.text = "Hallo von der Buch-VM!";
}
```

Nach einem Rebuild (den Befehl dazu erklärt Kapitel 7 im Detail – `nixos-rebuild switch` reicht für jetzt) zeigt `cat /etc/motd` den gesetzten Text. Lässt du `enable` weg, bleibt es beim Default `false` – `mkEnableOption` legt das automatisch so an – und dank `mkIf` passiert dann schlicht gar nichts, ganz ohne dass du das irgendwo explizit abfragen musst.

> 💡 **Nice to know:** `lib.types` bietet weit mehr als die Grundtypen – `listOf`, `attrsOf`, `nullOr`, `enum [ "a" "b" ]` und sogar verschachtelte `submodule`-Typen für strukturierte Optionen. Für den Einstieg brauchst du meist nur `str`, `int`, `bool` und `listOf`; der Rest lohnt sich, sobald deine eigenen Module komplexer werden.

> 💡 **Nice to know:** Jede `description`, die du einer `mkOption` mitgibst, ist keine Fleißarbeit für die Katz – sie taucht automatisch in `nixos-option` und in generierter Options-Dokumentation auf (bei Nixpkgs selbst z. B. unter search.nixos.org bzw. Anhang A des NixOS Manual). Gut dokumentierte eigene Module profitieren vom selben Mechanismus, wenn du sie mit denselben Tools rendern lässt.

## Typische Fehler

**1. Eine Option verwenden, die es gar nicht gibt:**

```
The option `services.httpd.enable' defined in `/etc/nixos/configuration.nix' does not exist.
```

*Ursache:* Tippfehler im Optionsnamen, oder das Modul, das diese Option deklariert, ist gar nicht eingebunden.
*Fix:* Namen mit `nixos-option` oder search.nixos.org gegenchecken; bei eigenen Modulen prüfen, ob sie überhaupt in `imports` stehen.

**2. Zwei Module setzen dieselbe nicht-mergbare Option unterschiedlich:**

```
The unique option `services.httpd.adminAddr' is defined multiple times, in `/etc/nixos/httpd.nix' and `/etc/nixos/configuration.nix'.
```

*Ursache:* Zwei Module definieren denselben Einzelwert (z. B. einen String) widersprüchlich; Nix kann nicht raten, welcher gemeint ist.
*Fix:* Eine der beiden Definitionen entfernen, oder bewusst mit `lib.mkForce` einen Gewinner festlegen.

(Beide Fehlermeldungen im Wortlaut aus dem NixOS Manual, Abschnitt "Modularity".)

## Übung

1. Schreib ein eigenes Modul mit einer neuen Option (Enable-Flag plus mindestens einer weiteren, typisierten Option), binde es in deine `configuration.nix` ein und rebuilde.
2. Provoziere absichtlich den zweiten Fehler oben: Setze `networking.hostName` in zwei verschiedenen, beide importierten Modulen auf unterschiedliche Werte, beobachte die Fehlermeldung – und behebe sie danach gezielt mit `lib.mkForce`.

**Lösungsskizze:**

Zu 1: Analog zum `begruessung.nix`-Beispiel oben.

Zu 2: Zwei Dateien mit je `networking.hostName = "a";` bzw. `networking.hostName = "b";`, beide in `imports` einbinden → Konfliktfehler beobachten → eine der beiden Zeilen durch `networking.hostName = lib.mkForce "a";` ersetzen → Fehler verschwindet.

## Zusammenfassung

- Module haben zwei Seiten: `options` (was gibt es?) und `config` (welcher Wert?) – die meisten Configs in freier Wildbahn sind reines `config`.
- `imports` bindet weitere Module ein; listenwertige Optionen werden zusammengeführt, einzelwertige können in Konflikt geraten.
- `mkIf` schaltet einen `config`-Block bedingt frei; `mkDefault`/`mkForce` regeln Priorität; `mkMerge` fasst mehrere `config`-Blöcke zusammen; `mkBefore` ordnet Listeneinträge.
- `lib` ist Nixpkgs' Standardbibliothek; für den Modulbau reichen anfangs `mkIf`/`mkDefault`/`mkForce`/`mkOption`/`mkEnableOption`.
- Enable-Flag per `mkEnableOption` plus `config = mkIf cfg.enable { ... };` ist das immer wiederkehrende Grundmuster eigener Module.
- Gute `description`-Texte zahlen sich über `nixos-option` und automatisch generierte Options-Doku aus.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `options`/`config`-Struktur, `mkBefore`, `mkForce`, `config`-Argument (Laziness), Merge-Verhalten | [NixOS Manual – Configuration Syntax, Modularity](https://nixos.org/manual/nixos/stable/) |
| Fehlermeldungen "does not exist" / "defined multiple times" | [NixOS Manual – Modularity](https://nixos.org/manual/nixos/stable/) (Wortlaut) |
| `mkIf`, `mkDefault`, `mkMerge`, `mkOption`, `mkEnableOption`, `lib.types` | Nixpkgs `lib`, dokumentiert im Nixpkgs Manual |
| `environment.etc.<name>.text` | NixOS-Option, siehe search.nixos.org |
