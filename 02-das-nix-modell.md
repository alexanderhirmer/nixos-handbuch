---
title: "Das Nix-Modell"
weight: 2
---

# Das Nix-Modell

## Lernziele

- Du kannst erklären, was der Nix Store ist und wie sich seine Pfade zusammensetzen.
- Du verstehst, was eine Derivation ist und wie sie sich vom fertigen Ergebnis unterscheidet.
- Du kennst Profile und Generationen und wie sie mit Rollback zusammenhängen.
- Du verstehst das Grundprinzip von Garbage Collection und GC-Roots.
- Du kannst erklären, wie eine Systemaktivierung technisch atomar abläuft.

## Warum das wichtig ist

Alles, was in den folgenden Kapiteln wie Magie aussieht – Rollback, parallele Paketversionen, reproduzierbare Builds – ist eigentlich nur die logische Konsequenz aus vier einfachen Bausteinen: Store, Derivation, Profil, GC-Root. Wer dieses Modell einmal verinnerlicht hat, muss `nixos-rebuild`, Fehlermeldungen und spätere Kapitel nicht mehr auswendig lernen, sondern kann sie herleiten.

## Der Nix Store

Fast alles, was Nix baut oder herunterlädt, landet unter `/nix/store/` – jedes Paket, jede Bibliothek, jede Konfigurationsdatei, sogar das System selbst. Ein typischer Pfad sieht so aus:

```
/nix/store/5mbglq5ldqld8sj57273aljwkfvj22mc-subversion-1.1.4
```

Der erste Teil des Namens ist ein 32 Zeichen langer, Base32-kodierter Hash, danach folgen Name und Version. Zwei Dinge daran sind entscheidend:

1. **Store-Pfade sind unveränderlich.** Einmal gebaut, wird ihr Inhalt nicht mehr angefasst. Deshalb sind sie – anders als `/usr/lib` bei klassischen Distributionen – nicht "der eine, aktuelle Ort" für eine Bibliothek, sondern einer von potenziell vielen. Firefox Version A und Version B können parallel existieren, ohne sich in die Quere zu kommen, weil sie unter verschiedenen Hashes liegen.
2. **Der Hash hängt von den Eingaben ab, nicht zwingend vom Ergebnis.** Im klassischen ("input-addressed") Modell, das NixOS standardmäßig verwendet, wird der Pfad-Hash aus der Bauanleitung und ihren Eingaben abgeleitet – nicht aus dem fertigen Inhalt. Zwei Builds, die bit-identischen Output produzieren, aber unterschiedlich dorthin gekommen sind, landen im Regelfall trotzdem unter unterschiedlichen Pfaden.

> ⚠️ Ungeprüft im Detail: Wie genau der Eingabe-Hash kryptographisch berechnet wird (Stichwort "hashDerivationModulo"), ist komplizierter, als es hier nötig ist zu wissen. Für den Alltag reicht das Modell "Hash = Funktion der Eingaben". Die exakte Formel steht im Nix-Referenzhandbuch.

Weil der Store für normale Prozesse effektiv nur lesbar ist, kannst du dort nichts einfach hineinschreiben:

```console
$ touch /nix/store/test
touch: cannot touch '/nix/store/test': Permission denied
```

Nur der Nix-Daemon legt neue Pfade an, und zwar ausschließlich über einen Build oder einen verifizierten Download.

## Derivations

Eine **Derivation** ist die Bauanleitung für einen Store-Pfad: welcher Builder wird mit welchen Argumenten aufgerufen, welche anderen Store-Pfade werden als Eingabe gebraucht, welche Ausgabe(n) entstehen. Derivationen sind selbst `.drv`-Dateien und liegen – wenig überraschend – ebenfalls im Store. Der Bau selbst läuft isoliert in einer Sandbox, damit nichts vom Host-System unkontrolliert hineinsickert, was die Reproduzierbarkeit brechen würde.

Zu jedem gebauten Pfad lässt sich die erzeugende Derivation nachschlagen:

```console
$ nix-store -q --deriver $(which bash)
/nix/store/<hash>-bash-5.x.drv
```

## Profile & Generationen

Ein **Profil** ist eine benannte, fortlaufende Kette von **Generationen** – im Grunde nichts anderes als nummerierte Symlinks, die jeweils auf einen konkreten Store-Pfad zeigen. Es gibt ein Systemprofil unter `/nix/var/nix/profiles/system` und pro Nutzer ein eigenes unter `/nix/var/nix/profiles/per-user/<name>/profile`, auf das `~/.nix-profile` verweist (Quelle: NixOS-Wiki, Nix Cookbook).

Jede erfolgreiche Systemaktivierung erzeugt einen neuen, durchnummerierten Link nach dem Muster:

```
/nix/var/nix/profiles/system-86-link
/nix/var/nix/profiles/system-87-link
/nix/var/nix/profiles/system-88-link
```

(Quelle: NixOS-Wiki; die konkreten Nummern sind natürlich pro Maschine unterschiedlich.) Rollback bedeutet im Kern nichts anderes, als das aktuelle Profil wieder auf einen älteren dieser Links zeigen zu lassen. Es gibt keine separate "Rollback-Historie" – nur Dateinamen mit Nummern. Wie du das konkret auslöst (`nixos-rebuild switch --rollback` und Verwandte), zeigt Kapitel 7.

## Garbage Collection & GC-Roots

Mit jedem Build und jeder neuen Generation sammeln sich im Store Pfade an, die niemand mehr aktiv braucht – alte Generationen, Zwischenschritte, verwaiste Abhängigkeiten. Die **Garbage Collection** räumt das auf, aber mit einer wichtigen Einschränkung: Sie löscht ausschließlich Pfade, die von keinem **GC-Root** aus mehr erreichbar sind.

GC-Roots sind unter anderem die aktuellen System- und Nutzerprofile samt ihrer Generationen-Links. Solange eine alte Generation noch als Root zählt, bleibt sie – und alles, was sie referenziert – erhalten, egal wie "veraltet" sie ist. Genau deshalb funktioniert Rollback zuverlässig: Der alte Zustand wurde nie wirklich weggeräumt, er wurde nur nicht mehr als "aktuell" markiert. Der eigentliche Aufräumbefehl und die NixOS-seitige Automatisierung (`nix.gc.*`) folgen in Kapitel 9 – hier zählt nur das Prinzip.

## Atomare Aktivierung – technisch

Zwei Symlinks unter `/run` sind für das Verständnis zentral: `/run/current-system` zeigt auf den Store-Pfad des aktuell aktiven Systems, `/run/booted-system` auf das, womit tatsächlich zuletzt gebootet wurde. Nach einem `switch` ohne Neustart können beide kurzzeitig auseinanderlaufen – ein nützliches Signal dafür, dass ein Reboot noch aussteht.

Der Ablauf beim Umschalten, vereinfacht:

1. Die neue Konfiguration wird **vollständig** gebaut – ein komplett neuer Store-Pfad entsteht, unabhängig vom bisherigen System.
2. Ein neuer, nummerierter Generation-Link wird im Systemprofil angelegt.
3. `/run/current-system` wird auf den neuen Pfad umgebogen.
4. Ein Aktivierungsskript gleicht den laufenden Zustand an (Dienste neu starten, Nutzer anlegen, …).

Schritt 3 ist ein einzelner Symlink-Tausch auf Dateisystemebene. Es gibt keinen Zwischenzustand, in dem `current-system` auf "halb altes, halb neues System" zeigt – entweder der alte Pfad oder der neue, nie etwas dazwischen.

## Vollständiges Beispiel

In deiner Buch-VM (die Live-Umgebung aus dem Vorwort reicht dafür bereits):

**1. Wohin zeigt das aktuelle System?**

```console
$ readlink /run/current-system
/nix/store/<hash>-nixos-system-nixos-26.05.XXXX-abcdef
```

Der genaue Hash ist bei dir anders – das ist genau der Punkt.

**2. Direkte vs. vollständige Abhängigkeiten von `bash`:**

```console
$ nix-store -q --references $(which bash)
```

zeigt nur die unmittelbaren Abhängigkeiten.

```console
$ nix-store -q --requisites $(which bash)
```

zeigt die komplette transitive Hülle – alles, was tatsächlich vorhanden sein muss, damit `bash` läuft (deutlich mehr Zeilen).

> 💡 **Nice to know:** Für Fälle, in denen der Standard-Hash-Ansatz stört – etwa wenn du zwei bit-identische Builds unbedingt unter demselben Pfad haben willst – gibt es das experimentelle Feature **content-addressed derivations**: Der Store-Pfad hängt dabei vom tatsächlichen Output ab statt von den Eingaben. Stand und Dokumentation im Nix-Referenzhandbuch unter den experimentellen Features.

> 💡 **Nice to know:** Wie "bit-für-bit-reproduzierbar" Nixpkgs tatsächlich ist, verfolgt das Community-Projekt [r13y.com](https://r13y.com) – ein guter Startpunkt, um zu sehen, dass Reproduzierbarkeit ein fortlaufendes Ziel ist, kein automatisches Naturgesetz.

## Typische Fehler

**1. Direktes Schreiben in den Store:**

```console
$ touch /nix/store/test
touch: cannot touch '/nix/store/test': Permission denied
```

*Ursache:* Normale Nutzer haben keine Schreibrechte auf `/nix/store`; nur der Nix-Daemon darf dort etwas anlegen.
*Fix:* Es gibt keinen – das ist Absicht. Wenn du etwas Neues brauchst, muss es über einen Build oder Download laufen, nicht per Hand.

**2. Ein kopiertes Binary läuft auf einer anderen Maschine nicht:**

```console
$ ls -la hello
-rwxr-xr-x 1 alex users 34816 Aug  9 10:00 hello
$ ./hello
bash: ./hello: No such file or directory
```

*Ursache:* Das Binary verweist auf einen fest einprogrammierten `/nix/store/…`-Pfad für seinen dynamischen Linker (und oft auch Bibliotheken). Existiert dieser Pfad auf der Zielmaschine nicht, meldet der Kernel "No such file or directory" – aber für den fehlenden Interpreter, nicht für die Datei selbst, die ja sichtbar vorhanden ist. Das verwirrt beim ersten Mal fast jeden.
*Fix:* Nicht nur die Binärdatei kopieren, sondern den vollständigen Closure (z. B. mit `nix copy` oder `nix-store --export`/`--import`), oder auf der Zielmaschine ebenfalls Nix/NixOS mit demselben Store-Pfad verwenden.

## Übung

1. Finde mit `readlink /run/current-system` den aktuellen Store-Pfad deines Systems heraus. Zähle den Hash-Teil – stimmen die 32 Zeichen aus diesem Kapitel?
2. Führe `nix-store -q --references $(which bash)` und `nix-store -q --requisites $(which bash)` aus. Worin unterscheidet sich die Ausgabe, und warum ist das so?

**Lösungsskizze:**

Zu 1: Ja, 32 Zeichen – das gilt für jeden gültigen Store-Pfad, unabhängig vom Paket.

Zu 2: `--references` listet nur die direkten Abhängigkeiten von `bash` selbst. `--requisites` listet die komplette transitive Hülle, also auch die Abhängigkeiten der Abhängigkeiten – deshalb ist diese Liste in aller Regel deutlich länger.

## Zusammenfassung

- Jeder Store-Pfad trägt einen 32-stelligen Hash im Namen, der im Standardmodell von den Build-Eingaben abhängt, nicht zwingend vom fertigen Inhalt.
- Derivationen (`.drv`-Dateien) sind die Bauanleitung, liegen selbst im Store und lassen sich mit `nix-store -q --deriver` nachschlagen.
- Profile sind benannte Ketten von Generationen (Symlinks); System- und Nutzerprofile liegen unter `/nix/var/nix/profiles`.
- Nur was von einem GC-Root aus erreichbar ist, übersteht eine Garbage Collection – deshalb bleiben alte Generationen samt Rollback-Fähigkeit erhalten, bis sie explizit aufgeräumt werden.
- `/run/current-system` zeigt atomar auf das aktive System; der Wechsel ist ein einzelner Symlink-Tausch, nie ein Zwischenzustand.
- Store-Pfade sind für normale Nutzer unveränderlich – Schreibversuche scheitern mit `Permission denied`.
- Binaries verlassen sich auf feste Store-Pfade zu ihren Abhängigkeiten; ohne vollständigen Closure kopiert, laufen sie auf einer anderen Maschine oft gar nicht.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| Store-Pfad-Format (32-stelliger Hash) | [Nix Reference Manual – nix-store --query](https://nix.dev/manual/nix/2.30/command-ref/nix-store/query.html) |
| `nix-store -q --references / --requisites / --deriver` | [Nix Reference Manual – nix-store --query](https://nix.dev/manual/nix/2.30/command-ref/nix-store/query.html) |
| `/nix/var/nix/profiles/system`, `system-<N>-link` | NixOS-Wiki – [Nix Cookbook](https://wiki.nixos.org/wiki/Nix_Cookbook) |
| `/run/current-system` | [nix-darwin-Projekt](https://github.com/nix-hackers/nix-darwin) (dokumentiert denselben NixOS-Mechanismus) |
| ENOENT bei kopierten Binaries ohne Closure | Allgemein dokumentiertes Nix-Verhalten (ELF-Interpreter-Pfad) |
