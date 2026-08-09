---
title: "Vorwort"
weight: 0
---

# Vorwort

## Lernziele

- Du kennst die Konventionen dieses Buchs und musst sie nicht bei jedem Kapitel neu erraten.
- Du weißt, auf welcher NixOS-Version das Buch basiert und was das für dich bedeutet.
- Du hast eine Wegwerf-VM mit NixOS am Laufen, in der du ab Kapitel 1 gefahrlos experimentieren kannst.
- Du weißt, wohin du dich wendest, wenn dieses Buch eine Frage nicht beantwortet.

## Warum das wichtig ist

NixOS-Fehlermeldungen sind am Anfang oft kryptisch, und ein falsch gesetztes Semikolon kann eine völlig andere Fehlermeldung produzieren, als man erwarten würde. Wer das zuerst an einer produktiv genutzten Maschine lernt, verliert Zeit und Nerven. Die Wegwerf-VM aus diesem Kapitel ist die Grundlage für praktisch jede Übung im Buch – sie einmal sauber aufzusetzen, erspart dir späteres Nachrüsten mitten in einem anderen Kapitel.

## Konventionen

Befehle mit vorangestelltem `#` brauchen Root-Rechte, Befehle mit `$` nicht – dieselbe Konvention wie im offiziellen NixOS-Handbuch ([Quelle](https://nixos.org/manual/nixos/stable/)).

- Codeblöcke sind immer mit ihrer Sprache markiert (`nix`, `bash`, `console`, `toml`). Wo es einen Unterschied macht, stehen Befehl und Ausgabe als zwei getrennte Blöcke.
- 💡-Boxen ("Nice to know") sind Zusatzstoff, kein Pflichtprogramm. Sie sollen Lust auf Themen machen, die dieses Buch nicht vollständig behandelt, und nennen einen Startpunkt zum Weiterlesen.
- ⚠️-Boxen ("Ungeprüft") markieren Stellen, an denen ich mir bei einer Aussage nicht zu 100 % sicher bin. Lieber eine ehrliche Lücke als eine erfundene Option.
- Jedes Kapitel endet mit einer Liste der verwendeten Befehle/Optionen samt Quelle.

## Zielgruppe

Vorausgesetzt wird solides Linux-Grundwissen: Shell, systemd, SSH, Partitionierung. Bei null anfängt dagegen Nix – weder die Sprache noch das Modulsystem noch der Store werden als bekannt vorausgesetzt. Beides kommt ab Kapitel 2 und 3.

## Versionsstand

Dieses Buch ist auf **NixOS 26.05 "Yarara"** zugeschnitten. Yarara wurde am 30. Mai 2026 offiziell veröffentlicht und erhält Bugfixes und Sicherheitsupdates bis zum 31. Dezember 2026 ([Release-Announcement](https://nixos.org/blog/announcements/2026/nixos-2605/)). Die Vorgängerversion 25.11 "Xantusia" gilt als deprecated.

Die nächste Stable-Version, 26.11, ist planmäßig für Ende November 2026 fällig. Solltest du dieses Buch danach lesen: Der grundsätzliche Weg bleibt in aller Regel gleich, aber einzelne Optionspfade können sich verschoben haben. Ein Beispiel dafür ist dir noch nicht begegnet, wird dir aber in einem späteren Kapitel begegnen: Das gescriptete Stage-1-Init gilt bereits jetzt als deprecated und soll mit 26.11 entfernt werden.

> ⚠️ Ungeprüft: Ob 26.11 zum Zeitpunkt, an dem du das hier liest, bereits erschienen ist, kann ich beim Schreiben nicht wissen. Prüfe im Zweifel [nixos.org/download](https://nixos.org/download/), welche Version dort aktuell als Stable geführt wird.

## Testumgebung: die Buch-VM

Für jede Übung in diesem Buch brauchst du eine NixOS-Maschine, die du bedenkenlos kaputt machen kannst. Zwei Wege dahin:

**Option A – VirtualBox (empfohlen für dieses Kapitel, plattformunabhängig).** Funktioniert gleich unter Linux, Windows und macOS und ist im offiziellen Handbuch dokumentiert.

**Option B – KVM/QEMU via libvirt.** Näher an einem echten Server, aber Linux-only als Host. Die genaue Provisionierung – inklusive Proxmox und LXC – behandelt Kapitel 4 ausführlich. Für dieses Vorwort reicht Option A, um startklar zu sein; wenn du ohnehin unter Linux arbeitest und lieber gleich mit libvirt übst, kannst du Kapitel 4 vorziehen und hierher zurückspringen.

Lade dir das **Minimal-ISO** für x86_64 von der offiziellen [Download-Seite](https://nixos.org/download/) herunter. Das Graphical-ISO enthält zusätzlich eine Live-Desktop-Umgebung, die du zum Üben nicht brauchst und die nur Downloadzeit kostet.

## Vollständiges Beispiel

**1. ISO herunterladen und Prüfsumme verifizieren.** Auf der Download-Seite liegt neben jedem Image ein SHA-256-Link. Nach dem Download beider Dateien in ein Verzeichnis:

```console
$ sha256sum -c nixos-minimal-x86_64-linux.iso.sha256
nixos-minimal-x86_64-linux.iso: OK
```

**2. VM in VirtualBox anlegen.** Nach dem offiziellen Handbuch:

- Neue Maschine, Typ "Linux / Other Linux"
- Base Memory: mindestens 768 MB laut Handbuch – für die Übungen in diesem Buch empfehle ich eher 2–4 GB, sonst bremsen einzelne Builds spürbar (das ist meine eigene Praxisempfehlung, keine offizielle Vorgabe)
- Neue Festplatte: mindestens 10 GB, für mehrere Generationen und Kapitel 9 (Store-Wartung) würde ich 20 GB ansetzen
- ISO über "Einstellungen → Massenspeicher" als CD/DVD einhängen
- Unter "Einstellungen → System → Prozessor": PAE/NX aktivieren
- Unter "Einstellungen → System → Beschleunigung": VT-x/AMD-V aktivieren
- Unter "Einstellungen → Display → Bildschirm": VMSVGA als Grafikcontroller wählen

**3. Booten und einloggen.** Am Bootmenü der Standardauswahl folgen (Enter beschleunigt). Nach dem Hochfahren bist du automatisch als Nutzer `nixos` eingeloggt, das Passwort ist leer:

```console
$ sudo -i
# ip a
```

Zeigt `ip a` eine Adresse außer `lo`, hat die VM per DHCP eine IP bekommen und ist netzwerkfähig – Voraussetzung für so ziemlich alles Weitere in diesem Buch.

> 💡 **Nice to know:** Community-Hilfe findest du im [NixOS Discourse](https://discourse.nixos.org), im Matrix-Room `#nix:nixos.org` oder im IRC-Kanal `#nixos` auf Libera.Chat – alle drei sind im offiziellen Handbuch als offizielle Anlaufstellen genannt. Bugs gehören in den [Nixpkgs-Issue-Tracker](https://github.com/NixOS/nixpkgs/issues).

> 💡 **Nice to know:** Nix, der Paketmanager, lässt sich auch auf einem beliebigen anderen Linux oder auf macOS installieren, ganz ohne NixOS – nützlich, wenn du reproduzierbare Dev-Umgebungen willst, ohne gleich das ganze Betriebssystem zu wechseln. Startpunkt: `curl -L https://nixos.org/nix/install | sh`, mehr dazu im Nix-Manual.

## Typische Fehler

**1. VirtualBox startet die VM nicht und meldet sinngemäß:**

```
VT-x is not available (VERR_VMX_NO_VMX)
```

*Ursache:* Hardware-Virtualisierung ist im Host-BIOS/UEFI deaktiviert, oder unter Windows kollidiert Hyper-V (z. B. durch WSL2 oder Windows-Sandbox aktiviert) mit VirtualBox' eigenem Hypervisor.
*Fix:* VT-x/AMD-V im BIOS/UEFI aktivieren; unter Windows ggf. Hyper-V-Features deaktivieren oder VirtualBox im Hyper-V-kompatiblen Modus betreiben.

**2. Die Prüfsumme nach dem Download passt nicht:**

```
nixos-minimal-x86_64-linux.iso: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
```

*Ursache:* Download abgebrochen oder beschädigt, seltener ein Problem mit dem gewählten Mirror.
*Fix:* Erneut herunterladen, Prüfsumme direkt und frisch von der Download-Seite kopieren statt aus altem Clipboard-Inhalt.

## Übung

1. Lege eine VirtualBox-VM nach den Vorgaben oben an, boote das Minimal-ISO und melde dich an. Prüfe mit `sudo -i` und `ip a`, dass die VM eine IP-Adresse hat.
2. Finde im Live-System heraus, mit welchem Befehl du dir Version und Revision des laufenden Systems anzeigen lassen kannst.

**Lösungsskizze:**

Zu 2: `nixos-version` zeigt einen String wie `26.05.XXXX.abcdef1234ab (Yarara)` – Versionsnummer, Build-Revision und Codename in einem.

## Zusammenfassung

- Root-Befehle stehen mit `#`, normale mit `$`; 💡 = Zusatzstoff, ⚠️ = unsichere Aussage.
- Buch-Basis ist NixOS 26.05 "Yarara", Support bis 31.12.2026, Nachfolger 26.11 planmäßig ab November 2026.
- Solides Linux-Wissen wird vorausgesetzt, Nix-Wissen ausdrücklich nicht.
- Die Übungs-VM lässt sich am schnellsten in VirtualBox aufsetzen (Minimum lt. Handbuch: 768 MB RAM, 10 GB Platte, VT-x/AMD-V + VMSVGA aktiv).
- Für Server-nahes Üben mit KVM/libvirt oder Proxmox: siehe Kapitel 4.
- Bei Problemen abseits dieses Buchs: NixOS Discourse, Matrix, `#nixos` auf Libera.Chat.
- Ab Kapitel 1 wird die hier aufgesetzte VM aktiv gebraucht.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `sha256sum -c` | Standard-Linux-Tool (coreutils) |
| `sudo -i` | Standard-Linux-Tool |
| `ip a` | Standard-Linux-Tool (iproute2) |
| `nixos-version` | NixOS-Systembefehl |
| VirtualBox-Setup (Memory/HDD/PAE-NX/VT-x/VMSVGA) | [NixOS Manual – Installing in a VirtualBox guest](https://nixos.org/manual/nixos/stable/) |
| Root-/Normal-User-Konvention (`#`/`$`) | [NixOS Manual – Preface](https://nixos.org/manual/nixos/stable/) |
