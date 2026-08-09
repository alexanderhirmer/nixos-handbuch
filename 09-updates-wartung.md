---
title: "Updates & Wartung"
weight: 9
---

# Updates & Wartung

## Lernziele

- Du kannst ein Release-Upgrade sicher vorbereiten und durchführen.
- Du weißt, wo die Release Notes stehen und wie du gezielt nach Breaking Changes filterst.
- Du setzt Garbage Collection manuell und automatisiert ein.
- Du verstehst Store-Optimierung und wann sie sich lohnt.
- Du erkennst die "volle `/boot`"-Falle und weißt, wie du sie vermeidest.
- Du nimmst Deprecation-Warnungen ernst, am konkreten Beispiel des gescripteten Stage 1.

## Warum das wichtig ist

Ein Update ist der Moment, in dem Rollback (Kapitel 2 und 7) tatsächlich gebraucht werden könnte – und Store-Pflege verhindert, dass dir trotz aller Reproduzierbarkeit irgendwann einfach die Festplatte vollläuft.

## Release-Upgrade: Ablauf

**Channel-Weg:**

```console
# nix-channel --add https://channels.nixos.org/nixos-26.05 nixos
# nixos-rebuild switch --upgrade
```

**Flake-Weg** (Kapitel 8): `inputs.nixpkgs.url` auf den neuen Branch ändern, dann

```console
$ nix flake update nixpkgs
# nixos-rebuild switch --flake
```

In beiden Fällen gilt dieselbe Vorsicht: Erst mit `nixos-rebuild test` (Kapitel 7) aktivieren statt direkt mit `switch`, idealerweise zuerst an einer unkritischen Maschine, mindestens einen vollständigen Reboot-Zyklus laufen lassen, bevor produktive Systeme folgen.

## Konkretes Beispiel: 25.11 → 26.05

Damit das nicht abstrakt bleibt, hier der tatsächlich aktuelle Breaking Change zwischen den beiden letzten Stable-Versionen: Mit 26.05 ist Stage 1 (das Initrd) standardmäßig systemd-basiert; die alte, gescriptete Variante ist deprecated und soll mit 26.11 entfernt werden. Ein temporärer Fallback existiert (`boot.initrd.systemd.enable = false;`), wird im Release-Notes-Text selbst aber ausdrücklich als "discouraged" bezeichnet.

Das ist kein rein kosmetischer Wechsel: Bekannt gewordene Probleme betreffen vor allem LUKS-verschlüsselte Root-Partitionen, die unter dem neuen systemd-Stage-1 teils eine explizite Device-Mapping-Konfiguration brauchen, die vorher automatisch lief (dokumentiert u. a. in einem offenen Nixpkgs-Issue). Genau deshalb lohnt sich vor einem Upgrade ein kurzer, gezielter Blick in die Release Notes – nicht danach, wenn die Maschine schon nicht mehr bootet.

Neu in 26.05 gibt es dafür zusätzlich ein eingebautes Sicherheitsnetz: sogenannte **Switch Inhibitors** vergleichen bestimmte Werte (z. B. die systemd-Version) zwischen alter und neuer Generation und verweigern den Wechsel, wenn sich etwas Kritisches unterscheidet – außer man erzwingt es explizit mit `NIXOS_NO_CHECK=1`. Genau die Art Vorsicht, die dieses Kapitel ohnehin empfiehlt, ist damit inzwischen teilweise schon eingebaut.

## Release Notes lesen

Die Release Notes stehen im NixOS Manual, Anhang B: `https://nixos.org/manual/nixos/stable/release-notes`. Der Aufbau ist immer gleich: neueste Version zuerst, konkrete Optionspfade werden benannt, und "deprecated"/"removed" (mit Zielversion) sind feste, wiederkehrende Begriffe. Praktischer Filtertipp: Nicht die komplette Liste lesen, sondern gezielt nach "deprecated", "removed" und "breaking" suchen (Browser-Suche reicht) – das sind die Einträge, die dich im schlimmsten Fall ein nicht bootendes System kosten, der Rest ist meist optional interessant.

## Garbage Collection: manuell & automatisch

Manuell, alles außer der aktuellen Generation entfernen:

```console
# nix-collect-garbage -d
4394 store paths deleted, 3467.28 MiB freed
```

Automatisiert, als NixOS-Option:

```nix
nix.gc = {
  automatic = true;
  dates = "weekly";
  options = "--delete-older-than 7d";
};
```

Das legt intern einen systemd-Timer an, der `nix-collect-garbage` nach genau diesem Zeitplan aufruft.

> ⚠️ Ein Nebeneffekt, der überrascht: Inhalte, die du z. B. über `builtins.fetchTarball` importierst und die sonst nirgends referenziert sind, zählen für die Garbage Collection als "nicht mehr gebraucht" und werden mitgelöscht – brauchst du sie erneut, lädt Nix sie einfach neu herunter. Kein Datenverlust, aber unter Umständen unerwartete Downloadzeit beim nächsten Build.

## Store-Optimierung

```console
# nix-store --optimise
```

sucht im gesamten Store nach Dateien mit identischem Inhalt und ersetzt Duplikate durch Hardlinks auf eine einzige Kopie – möglich, weil Store-Pfade unveränderlich sind (Kapitel 2). Das kann bei einem vollen Store einige Zeit dauern. Alternativ automatisch und inkrementell, bei jedem neuen Store-Pfad:

```nix
nix.settings.auto-optimise-store = true;
```

(Default: `false` – du entscheidest bewusst, ob sich der laufende Overhead für dich lohnt.)

## Die volle-`/boot`-Falle

Jede Generation bekommt nicht nur einen Bootmenü-Eintrag, sondern auch eine eigene Kopie von Kernel und Initrd – und die landen auf der (in Kapitel 4 bewusst klein gehaltenen) EFI-System-Partition, nicht im großzügig bemessenen `/nix/store`. Normale Garbage Collection allein hilft hier nicht zuverlässig, weil aktuelle Generationen als GC-Roots (Kapitel 2) genau davor geschützt sind, weggeräumt zu werden. Die gezielte Lösung begrenzt stattdessen direkt die Zahl der *behaltenen* Booteinträge:

```nix
boot.loader.systemd-boot.configurationLimit = 10;
# bzw. für GRUB:
# boot.loader.grub.configurationLimit = 10;
```

## Deprecations überleben

Die generelle Regel: Eine Deprecation-Warnung in den Release Notes ist keine Bitte, sondern eine Ankündigung mit Ablaufdatum – meist genau eine Release-Zyklus-Länge (ein halbes Jahr). Das systemd-Stage-1-Beispiel oben zeigt das Muster: neues Verhalten wird Standard, alte Variante bleibt eine Weile als Fallback verfügbar (hier `boot.initrd.systemd.enable = false;`), wird aber explizit als nicht empfohlener Übergang markiert, nicht als dauerhafte Lösung.

## Vollständiges Beispiel

Ein Wartungs-Setup, das die wichtigsten Punkte dieses Kapitels kombiniert:

```nix
{
  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 14d";
  };

  nix.settings.auto-optimise-store = true;

  boot.loader.systemd-boot.configurationLimit = 10;
}
```

Vor jedem größeren Upgrade zusätzlich manuell:

```console
$ # Release Notes gezielt auf "deprecated"/"removed" durchsuchen
# nixos-rebuild test --upgrade
# journalctl -b -p err   # Vorgriff Kapitel 11 – auf neue Fehler nach dem Boot prüfen
# nixos-rebuild switch --upgrade   # erst wenn test unauffällig war
```

> 💡 **Nice to know:** `nvd` und `nix-diff` sind Community-Tools, um zwei Generationen (oder zwei Derivationen) gegenüberzustellen und genau zu sehen, welche Pakete sich geändert haben – deutlich angenehmer als Release Notes und `list-generations` allein gegenzulesen.

> 💡 **Nice to know:** `system.autoUpgrade.enable = true;` (zusammen mit `system.autoUpgrade.dates`) lässt NixOS Upgrades komplett unbeaufsichtigt durchführen. Bequem für unkritische Maschinen, aber angesichts dieses Kapitels mit Bedacht einzusetzen – ein automatisches Upgrade liest die Release Notes nicht für dich.

## Typische Fehler

**1. System bootet nach dem 26.05-Upgrade nicht mehr (LUKS + systemd-Stage-1):**

```
Enter passphrase for [...]:
[...]
A start job is running for ...
```

– hängt endlos, kein Fortschritt, kein Fehlertext.

*Ursache:* Verschlüsselte Root-Partition, die unter dem neuen systemd-basierten Stage 1 eine explizite Device-Mapping-Konfiguration braucht, die vorher implizit funktioniert hat (dokumentierter, bekannter Fall).
*Fix:* Rollback zur vorherigen Generation über das Bootmenü (Kapitel 7); anschließend die LUKS-relevanten Abschnitte der Release Notes lesen, bevor der nächste Versuch startet. Temporär hilft auch `boot.initrd.systemd.enable = false;` – laut Release Notes selbst aber nur als Übergangslösung gedacht.

**2. Bootloader-Installation schlägt mit vollem Datenträger fehl:**

```
No space left on device
```

*Ursache:* Viele Generationen ohne `configurationLimit` haben die kleine EFI-System-Partition mit Kernel-/Initrd-Kopien gefüllt – unabhängig davon, wie viel Platz auf `/nix/store` noch frei ist.
*Fix:* `boot.loader.systemd-boot.configurationLimit` (oder das GRUB-Äquivalent) setzen, dann `nix-collect-garbage -d`.

## Übung

1. Öffne die Release Notes (Anhang B) und identifiziere mindestens einen weiteren "deprecated"-Eintrag neben dem Stage-1-Beispiel aus diesem Kapitel.
2. Übernimm die drei Optionen aus dem Wartungs-Beispiel oben (`nix.gc`, `auto-optimise-store`, `configurationLimit`) in deine Buch-VM, rebuild, und prüfe mit `nixos-rebuild list-generations`, wie viele Generationen aktuell existieren.

**Lösungsskizze:**

Zu 1: Individuell, abhängig davon, welche Release Notes gerade aktuell sind.

Zu 2: Nach dem Rebuild sollte `list-generations` unverändert alle bisherigen Generationen zeigen – `configurationLimit` und `nix.gc` wirken erst beim nächsten Bootloader-Update bzw. beim nächsten GC-Lauf, nicht rückwirkend sofort.

## Zusammenfassung

- Release-Upgrades laufen über `nix-channel`+`--upgrade` oder über `nix flake update`+`--flake`; in beiden Fällen zuerst `test`, dann erst `switch`.
- Das aktuell relevanteste Beispiel: systemd-Stage-1 ist seit 26.05 Standard, die alte gescriptete Variante deprecated bis 26.11 – mit bekannten LUKS-Fallstricken.
- Release Notes stehen in Anhang B des Manual; gezielt nach "deprecated"/"removed"/"breaking" suchen statt alles zu lesen.
- `nix-collect-garbage -d` räumt manuell auf, `nix.gc.automatic` automatisiert das über einen systemd-Timer.
- `nix-store --optimise` bzw. `nix.settings.auto-optimise-store` sparen Platz durch Hardlinks auf identische Dateien.
- Eine volle `/boot`-Partition ist ein eigenes Problem, das normale GC nicht löst – `configurationLimit` begrenzt gezielt die Zahl der behaltenen Booteinträge.
- Deprecation-Warnungen haben ein Ablaufdatum; der angebotene Fallback ist eine Übergangslösung, kein Dauerzustand.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `nix-channel --add/--upgrade`, `nixos-rebuild switch --upgrade` | [NixOS Manual – Upgrading NixOS](https://nixos.org/manual/nixos/stable/) |
| `boot.initrd.systemd.enable`, Switch Inhibitors, systemd-Stage-1-Deprecation | [NixOS Manual – Release Notes (Anhang B)](https://nixos.org/manual/nixos/stable/release-notes) |
| LUKS/systemd-Stage-1-Bootproblem | [Nixpkgs-Issue #527478](https://github.com/nixos/nixpkgs/issues/527478) |
| `nix-collect-garbage -d`, Beispielausgabe | [NixOS-Wiki – Storage optimization](https://wiki.nixos.org/wiki/Storage_optimization) |
| `nix.gc.automatic`/`dates`/`options` | [NixOS-Wiki – Storage optimization](https://wiki.nixos.org/wiki/Storage_optimization), [NixOS & Flakes Book](https://nixos-and-flakes.thiscute.world/) |
| `nix-store --optimise`, `nix.settings.auto-optimise-store` | [Nix Reference Manual](https://nix.dev/manual/nix/2.26/command-ref/new-cli/nix3-store-optimise), [MyNixOS](https://mynixos.com/nixpkgs/option/nix.settings.auto-optimise-store) |
| `boot.loader.systemd-boot.configurationLimit`/`grub.configurationLimit` | [NixOS & Flakes Book – Other Useful Tips](https://nixos-and-flakes.thiscute.world/nixos-with-flakes/other-useful-tips) |
| `nvd`, `nix-diff` | Community-Tools, siehe jeweilige Projekt-Repositories |
| `system.autoUpgrade` | NixOS-Option, siehe search.nixos.org |
