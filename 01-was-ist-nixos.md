---
title: "Was ist NixOS?"
weight: 1
---

# Was ist NixOS?

## Lernziele

- Du kannst erklären, was "Config-Drift" ist und warum klassisches Paketmanagement es begünstigt.
- Du verstehst den Unterschied zwischen deklarativer und imperativer Systemkonfiguration.
- Du kennst NixOS' Kernversprechen: atomare Upgrades und triviale Rollbacks.
- Du kannst einschätzen, wann sich NixOS für dich NICHT lohnt.
- Du kannst Nix, Nixpkgs, NixOS und Home Manager auseinanderhalten.

## Warum das wichtig ist

Bevor du Zeit in eine neue Sprache und ein neues Modulsystem investierst, lohnt sich die Frage, welches Problem das eigentlich löst – und ob du dieses Problem überhaupt hast. Dieses Kapitel liefert die Landkarte, damit die Konzepte in Kapitel 2 und 3 nicht im luftleeren Raum stehen.

## Das Problem: Config-Drift

Stell dir einen Server vor, der seit drei Jahren läuft. Ein Kollege hat mal ein Paket per `apt install` nachinstalliert, um einen Bug zu debuggen, und danach vergessen, es wieder zu entfernen. Jemand anderes hat eine Konfigurationsdatei von Hand angepasst, weil "das schnell ging". Ein drittes Paket wurde auf eine bestimmte Version gepinnt, weil ein Update damals etwas kaputt gemacht hat – warum genau, weiß niemand mehr.

Keiner dieser Schritte war für sich falsch. Aber in Summe ist der Server jetzt ein Unikat, dessen tatsächlicher Zustand nirgends vollständig dokumentiert ist. Genau das nennt man **Config-Drift**: Der reale Zustand eines Systems entfernt sich schleichend von jeder schriftlich fixierten Beschreibung, weil klassisches Paketmanagement (dpkg, rpm, apt, yum) grundsätzlich *imperativ* funktioniert – du sagst dem System "installiere jetzt X", nicht "so soll das System aussehen". Jeder Befehl verändert den laufenden Zustand, ohne dass es einen eingebauten Mechanismus gibt, der garantiert, dass zwei so administrierte Server tatsächlich identisch sind. "Works on my machine" ist die logische Konsequenz.

## Deklarativ vs. imperativ

NixOS dreht das Prinzip um: Du beschreibst in einer einzigen Konfiguration, wie das System aussehen *soll* – welche Pakete installiert sind, welche Dienste laufen, welche Nutzer existieren. Ein Werkzeug (`nixos-rebuild`, siehe Kapitel 7) sorgt dafür, dass der tatsächliche Zustand exakt dieser Beschreibung entspricht. Läuft ein Paket bereits in der richtigen Version, passiert nichts. Fehlt es, wird es gebaut oder heruntergeladen. Wird es aus der Konfiguration entfernt, verschwindet es beim nächsten Rebuild aus dem aktiven System.

Der entscheidende Unterschied: Die Konfigurationsdatei *ist* die Quelle der Wahrheit, nicht die Historie der Befehle, die irgendwann mal ausgeführt wurden. Zwei Maschinen mit derselben Konfiguration sind – von echter Hardware-Varianz abgesehen – funktional identisch. Das ist der Kern dessen, was NixOS unter Reproduzierbarkeit versteht; wie das technisch funktioniert (Store, Derivations, Hashes), erklärt Kapitel 2 im Detail.

## Atomare Upgrades & Rollback

Ein `nixos-rebuild switch` baut die neue Konfiguration komplett fertig, *bevor* überhaupt etwas am laufenden System verändert wird. Erst wenn der Build vollständig durch ist, wird umgeschaltet – und zwar in einem Schritt. Es gibt keinen Zwischenzustand, in dem die Hälfte der neuen Pakete da ist und die andere Hälfte fehlt. Schlägt der Build fehl, läuft das System einfach mit der alten, unveränderten Konfiguration weiter.

Jede erfolgreiche Aktivierung erzeugt eine neue **Generation** – eine Art Systemzustand als benannter Boot-Eintrag. Alte Generationen bleiben verfügbar, bis sie explizit aufgeräumt werden (Kapitel 9). Läuft etwas schief, das erst nach dem Neustart auffällt, wählst du im Bootmenü einfach die vorherige Generation – kein Wiederherstellen aus dem Backup, kein Neuaufsetzen. Die genaue Mechanik dahinter zeigt Kapitel 7.

## Wo sich NixOS nicht lohnt

Ehrlich zu sein gehört hier dazu:

- **Echte Zeitnot.** Die Lernkurve ist real. Wenn ein Server heute stehen muss und niemand im Team je mit Nix gearbeitet hat, ist "schnell mit Ansible/Bash draufinstallieren" oft die pragmatischere Wahl.
- **Wirklich kurzlebige Wegwerf-Maschinen.** Für eine VM, die in einer Stunde wieder weg ist, lohnt sich die deklarative Denkarbeit selten.
- **Software mit nicht-FHS-konformem dynamischem Linking.** Manche proprietären Programme erwarten Bibliotheken an fest einprogrammierten Pfaden wie `/lib` oder `/usr/lib`, die es auf NixOS in dieser Form nicht gibt. Das lässt sich mit `buildFHSEnv` und ähnlichen Werkzeugen lösen (Kapitel 12), ist aber zusätzlicher Aufwand, den andere Distributionen nicht verlangen.
- **Teams, die kollektiv keine Nix-Sprache lernen wollen.** Deklarative Konfiguration nützt nur, wenn auch wirklich alle sie pflegen – ein System, das nur eine Person versteht, hat das Config-Drift-Problem nur verschoben.

## Einordnung: Nix, Nixpkgs, NixOS, Home Manager

Die vier Begriffe werden oft synonym benutzt, sind aber unterschiedliche Dinge:

- **Nix** ist der Paketmanager und die zugrundeliegende Sprache/Build-Engine. Läuft auf so gut wie jedem Linux und auf macOS – auch ganz ohne NixOS.
- **Nixpkgs** ist die riesige Sammlung von Paketdefinitionen ("Rezepten"), die Nix baut. Wenn du `pkgs.irgendwas` schreibst, kommt das aus Nixpkgs.
- **NixOS** ist die komplette Linux-Distribution, bei der *auch die Systemkonfiguration selbst* – Dienste, Nutzer, Netzwerk, Kernel-Parameter – als Nix-Ausdruck beschrieben wird. Darum geht es in diesem Buch.
- **Home Manager** ist ein eigenständiges, community-getragenes Projekt, das denselben deklarativen Ansatz auf Nutzer-Ebene anwendet (Dotfiles, Shell-Konfiguration). Es ist kein Teil von NixOS und auch nicht zwingend erforderlich – mehr dazu in Kapitel 13.

## Vollständiges Beispiel

So sieht eine minimale, aber vollständige NixOS-Konfiguration aus, die den OpenSSH-Server aktiviert (Beispiel aus dem offiziellen Handbuch):

```nix
# /etc/nixos/configuration.nix
{ config, pkgs, ... }:
{
  imports = [
    ./hardware-configuration.nix
  ];

  boot.loader.grub.device = "/dev/sda";   # nur für BIOS-Systeme
  boot.loader.systemd-boot.enable = true; # nur für UEFI-Systeme

  services.openssh.enable = true;
}
```

Der Unterschied zum imperativen Weg lässt sich so zusammenfassen: Auf einer klassischen Distribution würdest du etwas wie `apt install openssh-server && systemctl enable ssh` ausführen – ein Befehl, der irgendwann in der Server-Historie verschwindet und auf einer zweiten Maschine erneut (und vielleicht leicht anders) ausgeführt werden müsste. Hier steht der gewünschte Zustand stattdessen in einer Datei, die sich versionieren, diffen und auf beliebig viele Maschinen anwenden lässt.

> 💡 **Nice to know:** Eine von der Community gepflegte, nicht-abschließende Liste von Firmen, die Nix oder NixOS produktiv einsetzen, findest du im GitHub-Repo [ad-si/nix-companies](https://github.com/ad-si/nix-companies). Für kommerziellen Support gibt es außerdem eine offizielle [Liste auf nixos.org](https://nixos.org/community/commercial-support/).

> 💡 **Nice to know:** NixOS ist nicht die einzige Distribution mit diesem Ansatz. [GNU Guix](https://guix.gnu.org) verfolgt eine sehr ähnliche Philosophie (ebenfalls funktional, ebenfalls mit eigenem Store), nutzt aber Guile Scheme statt der Nix-Sprache. Wer die Konzepte hier mag, findet dort vieles wieder.

## Typische Fehler

**1. Reflexartig einen fremden Paketmanager benutzen:**

```
$ apt install htop
bash: apt: command not found
```

*Ursache:* NixOS hat kein apt, dpkg, yum oder Ähnliches. Pakete laufen ausschließlich über Nix-Mechanismen.
*Fix:* `nix-env -iA nixos.htop` für den Ad-hoc-Weg (Kapitel 6) oder – empfohlen – das Paket in `environment.systemPackages` eintragen und `nixos-rebuild switch` ausführen.

**2. Eine generierte Konfigurationsdatei direkt bearbeiten wollen:**

```
$ vim /etc/ssh/sshd_config
...
E212: Can't open file for writing
```

*Ursache:* Viele von NixOS verwaltete Dateien unter `/etc` sind Symlinks in den (read-only) Nix Store. Manuelle Änderungen sind gar nicht erst möglich – und selbst wenn, würden sie beim nächsten Rebuild überschrieben.
*Fix:* Die zugehörige Option in `configuration.nix` setzen (z. B. `services.openssh.settings.PermitRootLogin`) und neu bauen.

## Übung

1. Führe in deiner Buch-VM aus dem Vorwort `apt install irgendwas` aus und beobachte die Fehlermeldung. Was sagt dir das über NixOS' Grundphilosophie?
2. Überlege: In welcher Situation aus deinem eigenen Arbeitsalltag hätte dir ein Rollback wie in diesem Kapitel beschrieben schon geholfen – und in welcher wäre der ganze Aufwand eher unnötig gewesen?

**Lösungsskizze:**

Zu 1: Die Fehlermeldung `command not found` zeigt direkt, dass NixOS bewusst keinen zweiten, parallelen imperativen Paketmanager neben Nix pflegt – es gibt nur den einen, deklarativen Weg (plus den Ad-hoc-Weg über `nix-env`, der aber ebenfalls über Nix läuft, nicht über ein separates System).

Zu 2: Individuelle Antwort, es gibt hier keine pauschal richtige Lösung – es geht ums Abwägen von Lernaufwand gegen den Nutzen von Reproduzierbarkeit und Rollback.

## Zusammenfassung

- Config-Drift entsteht, weil klassisches Paketmanagement imperativ ist: Befehle verändern Zustand, ohne dass eine zentrale Beschreibung mitgeführt wird.
- NixOS beschreibt den gewünschten Systemzustand deklarativ in einer Konfiguration; ein Werkzeug gleicht die Realität daran an.
- Upgrades sind atomar: entweder komplett oder gar nicht, nie halb.
- Jede erfolgreiche Aktivierung erzeugt eine Generation; Rollback heißt einfach "vorherige Generation booten".
- NixOS lohnt sich nicht immer – Zeitdruck, echte Wegwerf-Maschinen und nicht-FHS-konforme Software sind reale Gegenargumente.
- Nix (Paketmanager), Nixpkgs (Paketsammlung), NixOS (Distribution) und Home Manager (Nutzer-Ebene) sind vier verschiedene, aber zusammenhängende Dinge.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| Minimalbeispiel `configuration.nix` mit `services.openssh.enable` (`services.sshd.enable` ist laut NixOS-Optionsdatenbank ein Alias dafür) | [NixOS Manual – Installation](https://nixos.org/manual/nixos/stable/) |
| `apt install` (Vergleichsbeispiel, nicht NixOS-spezifisch) | Allgemeines Linux-Wissen |
| Vim-Fehler `E212` | Vim-Dokumentation (`:help E212`) |
