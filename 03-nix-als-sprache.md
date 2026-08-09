---
title: "Nix als Sprache"
weight: 3
---

# Nix als Sprache

## Lernziele

- Du kennst die grundlegenden Werttypen von Nix: Strings, Zahlen, Booleans, Pfade, Listen, Attribute Sets.
- Du kannst einfache und curried Funktionen schreiben und anwenden.
- Du verstehst `let/in`, `rec` und `with` – und kennst die Scoping-Falle von `with`.
- Du kannst `import` von der modulsystem-eigenen `imports`-Option unterscheiden.
- Du kannst erklären, was Lazy Evaluation praktisch für dein Konfigurations-File bedeutet.

## Warum das wichtig ist

Jede `configuration.nix`, jedes Modul, jedes Flake ist am Ende nichts als ein Nix-Ausdruck. Ohne die Grundlagen aus diesem Kapitel liest sich das Modulsystem in Kapitel 5 wie Magie – mit ihnen wird es vorhersagbar. Du brauchst hier keine vollständige Programmiersprachen-Referenz, nur so viel, dass du fremden Code lesen und eigene kleine Module schreiben kannst.

## Grundsyntax

Nix kennt wenige, aber konsequent genutzte Werttypen:

```nix
42                  # Ganzzahl
3.14                # Fließkommazahl
true                # Boolean
null                # Abwesenheit eines Werts
"ein String"        # String
./relativer/pfad     # Pfad (eigener Typ, keine Zeichenkette!)
/absoluter/pfad      # ebenfalls ein Pfad
[ 1 2 3 ]           # Liste – Elemente durch Leerzeichen getrennt, NICHT durch Komma
{ a = 1; b = 2; }   # Attribute Set – Name/Wert-Paare, jedes mit Semikolon beendet
```

Zwei Stolpersteine für alle, die von JSON oder anderen Sprachen kommen: Listen werden mit Leerzeichen getrennt, nicht mit Kommas, und jede Definition in einem Attribute Set braucht ihr eigenes Semikolon – auch die letzte.

**Pfade** sind ein eigener Werttyp, keine Strings. `./foo.nix` wird beim Parsen relativ zum Ort der aktuellen Datei aufgelöst, nicht relativ zum aktuellen Arbeitsverzeichnis der Shell.

**String-Interpolation** fügt Ausdrücke in einen String ein:

```nix
let name = "Welt"; in "Hallo, ${name}!"
```

ergibt `"Hallo, Welt!"`.

**Attribute Sets lassen sich verschachteln**, und Punkte in Namen sind Kurzschreibweise dafür:

```nix
{ services.httpd.enable = true; }
# ist dasselbe wie:
{ services = { httpd = { enable = true; }; }; }
```

(Dieses Beispiel kennst du sinngemäß schon aus Kapitel 1 – dort war es einfach nur schon in eine echte NixOS-Konfiguration eingebettet.)

## Funktionen & Currying

Eine Funktion mit einem Parameter:

```nix
x: x + 1
```

Anwendung erfolgt durch Nebeneinanderstellen, ohne Klammern und ohne Komma:

```nix
(x: x + 1) 41    # ergibt 42
```

Mehrere Parameter gibt es technisch nicht – stattdessen currying: Eine Funktion mit "zwei Parametern" ist eigentlich eine Funktion, die eine Funktion zurückgibt:

```nix
let
  addiere = a: b: a + b;
  addiere5 = addiere 5;   # Teilanwendung – addiere5 ist selbst wieder eine Funktion
in
addiere5 10   # ergibt 15
```

**Attribute-Set-Pattern.** Der wichtigste Funktionstyp in NixOS-Configs ist die, die ein einzelnes Attribute Set als Argument nimmt und dabei gleich destrukturiert:

```nix
{ config, pkgs, ... }:
{
  services.openssh.enable = true;
}
```

Genau das war die erste Zeile im Beispiel aus Kapitel 1. Jetzt kannst du sie lesen: Das ist eine Funktion mit *einem* Argument (einem Attribute Set), aus dem `config` und `pkgs` herausgezogen werden; `...` bedeutet "im Set dürfen noch weitere Attribute stehen, die interessieren hier nicht". Parameter können auch Standardwerte bekommen (`{ config, pkgs, wichtig ? "default", ... }:`), und mit `args@{ config, ... }:` bekommst du zusätzlich das komplette Set unter dem Namen `args`.

## `let`/`in`, `rec` und `with`

**`let/in`** bindet lokale Namen für einen nachfolgenden Ausdruck:

```nix
let
  x = 1;
  y = 2;
in
x + y
```

**`rec`** macht ein Attribute Set intern selbstreferenzierend. Ohne `rec` sehen sich Attribute innerhalb desselben Sets nicht gegenseitig:

```nix
{ a = 1; b = a + 1; }   # Fehler – a ist hier nicht sichtbar
rec { a = 1; b = a + 1; }   # funktioniert, b wird 2
```

**`with`** holt alle Namen eines Attribute Sets in den Sichtbereich des nachfolgenden Ausdrucks, um Wiederholung zu vermeiden:

```nix
let farben = { rot = "#ff0000"; gruen = "#00ff00"; }; in
with farben; [ rot gruen ]
```

> ⚠️ **Die Scoping-Falle von `with`:** Wenn zwei `with`-Quellen (oder ein `with` und eine umschließende Variable) denselben Namen definieren, ist beim Lesen des Codes nicht mehr offensichtlich, woher ein bloßer Bezeichner eigentlich kommt – und im Zweifel gewinnt eine Auflösungsreihenfolge, die man sich nicht immer merkt. Genau deshalb raten Nixpkgs-Styleguides dazu, `with` nicht wild zu verschachteln oder mit `let`-Bindungen gleichen Namens zu mischen.

## `import` – und warum das nicht dasselbe wie `imports` ist

`import` ist eine eingebaute Funktion: Sie nimmt einen Pfad und wertet die dort stehende Nix-Datei als Ausdruck aus.

```nix
import ./hilfsfunktionen.nix
```

lädt `hilfsfunktionen.nix` und gibt zurück, was darin steht. Enthält die Datei selbst eine Funktion, rufst du sie direkt weiter auf:

```nix
import ./hilfsfunktionen.nix { }
```

Das ist auch das Muster hinter `import <nixpkgs> { }`, mit dem man sich das komplette Nixpkgs-Set in den Sichtbereich holt.

Das ist etwas grundlegend anderes als die `imports = [ ./vpn.nix ./kde.nix ];`-Zeile, die du in NixOS-Modulen siehst. `import` (Singular) ist Sprachfeature – funktioniert überall, immer gleich. `imports` (Plural) ist eine ganz normale Options-Definition, die nur deshalb etwas Besonderes bewirkt, weil das *Modulsystem* diesen einen Namen speziell behandelt und die gelisteten Dateien selbst einliest und zusammenführt. Die Mechanik dahinter gehört zu Kapitel 5, nicht zur Sprache selbst – aber die Namensähnlichkeit sorgt zuverlässig für Verwirrung, deshalb an dieser Stelle schon der Hinweis.

## Lazy Evaluation praktisch

Nix wertet nur aus, was tatsächlich gebraucht wird. Das erklärt unter anderem, warum `config` in einem NixOS-Modul als Funktionsargument hereinkommt, obwohl `config` selbst das *Ergebnis* der Zusammenführung aller Module ist – solange kein Modul während seiner eigenen Auswertung auf genau den Teil von `config` zugreift, den es selbst erzeugt, geht das auf, ohne in einer Endlosschleife zu landen.

Praktisch bedeutet das auch: Ein fehlerhafter Ausdruck irgendwo in deiner Konfiguration muss nicht sofort auffallen – erst wenn tatsächlich jemand darauf zugreift, schlägt die Auswertung fehl. Das vollständige Beispiel unten zeigt das direkt.

## Vollständiges Beispiel

Zwei Dateien im selben Verzeichnis:

```nix
# hilfsfunktionen.nix
{
  begruessung = anrede: name: "${anrede}, ${name}!";
}
```

```nix
# beispiel.nix
let
  hilfe = import ./hilfsfunktionen.nix;

  person = rec {
    vorname = "Ada";
    nachname = "Lovelace";
    vollname = vorname + " " + nachname;
  };

  farben = {
    rot = "#ff0000";
    gruen = "#00ff00";
    blau = "#0000ff";
  };
in
{
  gruss = hilfe.begruessung "Hallo" person.vollname;
  farbliste = with farben; [ rot gruen blau ];
  landmine = 1 / 0;   # wird nie berechnet, solange niemand darauf zugreift
}
```

Ausgewertet mit `nix-instantiate` (kein `nix repl` nötig, siehe Nice-to-know unten):

```console
$ nix-instantiate --eval -E '(import ./beispiel.nix).gruss'
"Hallo, Ada Lovelace!"

$ nix-instantiate --eval -E '(import ./beispiel.nix).farbliste'
[ "#ff0000" "#00ff00" "#0000ff" ]

$ nix-instantiate --eval -E '(import ./beispiel.nix).landmine'
error: division by zero
```

Der letzte Aufruf beweist die Lazy Evaluation ganz konkret: Solange du `landmine` nicht explizit abfragst, wertet das gesamte `beispiel.nix` anstandslos aus – der Fehler schlummert einfach ungenutzt im Set.

> 💡 **Nice to know:** Für interaktives Ausprobieren gibt es `nix repl` – eine Read-Eval-Print-Loop für genau solche Experimente, ohne jedes Mal eine Datei anzulegen. Volle Einführung inklusive Debugging-Workflow folgt in Kapitel 11; nichts spricht aber dagegen, es dir schon jetzt parallel zu diesem Kapitel anzusehen.

> 💡 **Nice to know:** Damit Formatierungsdiskussionen erst gar nicht aufkommen, gibt es automatische Formatter für Nix-Code – [nixpkgs-fmt](https://github.com/nix-community/nixpkgs-fmt) (älter, in Nixpkgs selbst genutzt) und [alejandra](https://github.com/kamadorueda/alejandra) (neuer, aktiver entwickelt). Beide sind reine Formatierer ohne Konfigurationsoptionen – bewusst, um genau die Art von Stildebatten zu vermeiden, die andere Sprachen plagen.

## Typische Fehler

**1. Fehlendes Semikolon:**

```nix
{
  a = 1
  b = 2;
}
```

```
error: syntax error, unexpected identifier, expecting ';'
```

*Ursache:* Jede Attribut-Definition braucht ihr eigenes abschließendes Semikolon, auch wenn direkt danach schon die nächste beginnt.
*Fix:* Semikolon ergänzen – `a = 1;`.

**2. Selbstbezug ohne `rec` vergessen:**

```nix
{ a = 1; b = a + 1; }
```

```
error: undefined variable 'a'
```

*Ursache:* In einem normalen (nicht-rekursiven) Attribute Set sehen sich die Attribute untereinander nicht. `a` wird stattdessen in der *umschließenden* Umgebung gesucht – und dort existiert es nicht.
*Fix:* `rec { a = 1; b = a + 1; }`, wenn Selbstbezug tatsächlich gewollt ist.

## Übung

1. Schreib eine curried Funktion `multipliziere`, die zwei Zahlen multipliziert, und werte `multipliziere 6 7` per `nix-instantiate --eval` aus.
2. Bau dir – analog zum Kapitel-Beispiel – ein Attribute Set mit einer harmlosen und einer "Landmine"-Definition (z. B. einer Division durch 0). Zeig dir selbst, dass sich das ganze Set trotzdem auswerten lässt, solange du gezielt nur das harmlose Attribut abfragst.

**Lösungsskizze:**

Zu 1:
```console
$ nix-instantiate --eval -E 'let multipliziere = a: b: a * b; in multipliziere 6 7'
42
```

Zu 2: Genau das Verhalten, das `landmine` im Kapitel-Beispiel oben zeigt – Zugriff auf das harmlose Attribut klappt, Zugriff auf die Landmine wirft `error: division by zero`.

## Zusammenfassung

- Nix kennt Strings, Zahlen, Booleans, `null`, Pfade (eigener Typ!), Listen (leerzeichengetrennt) und Attribute Sets (semikolongetrennt).
- Funktionen nehmen technisch immer nur einen Parameter; "mehrere Parameter" ist Currying. Das `{ config, pkgs, ... }:`-Muster destrukturiert ein Attribute-Set-Argument.
- `rec` macht ein Attribute Set selbstreferenzierend; ohne `rec` sehen sich Geschwister-Attribute nicht.
- `with` holt Namen eines Sets in den Sichtbereich – praktisch, aber bei Überschneidungen eine Quelle für unklaren Code.
- `import` (Sprachfeature) und `imports` (Modulsystem-Option) sind zwei verschiedene Dinge, die nur zufällig ähnlich heißen.
- Lazy Evaluation bedeutet: Nicht abgefragte Werte werden nicht berechnet – auch nicht, wenn sie fehlerhaft sind.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| Attribute-Set-Syntax, Listen, `let/in`, Funktionen, `with`-Beispiel | [NixOS Manual – Configuration Syntax](https://nixos.org/manual/nixos/stable/) |
| `import <nixpkgs> { }`-Muster | [NixOS Manual – Adding Custom Packages](https://nixos.org/manual/nixos/stable/) |
| `nix-instantiate --eval` | Nix Reference Manual (Command-Line-Referenz) |
| `nixpkgs-fmt` | https://github.com/nix-community/nixpkgs-fmt |
| `alejandra` | https://github.com/kamadorueda/alejandra |
