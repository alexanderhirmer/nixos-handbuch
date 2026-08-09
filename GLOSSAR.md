# Glossar

Laufend gepflegt, ein Eintrag pro eingeführtem Begriff. Vollversion – der Anhang im fertigen Buch kompiliert daraus eine kuratierte Kurzfassung.

## Kapitel 0 – Vorwort

- **Minimal-ISO**: NixOS-Installationsmedium ohne grafische Oberfläche, bootet direkt auf die Kommandozeile. Kleiner als das Graphical-ISO.
- **Graphical-ISO**: NixOS-Installationsmedium mit Live-Desktop und grafischem Installer.
- **`nixos-version`**: Befehl, der Versionsnummer, Build-Revision und Codename des laufenden Systems anzeigt.

## Kapitel 1 – Was ist NixOS?

- **Config-Drift**: Der reale Zustand eines Systems entfernt sich schleichend von jeder schriftlich fixierten Beschreibung, weil imperatives Paketmanagement keinen zentralen Soll-Zustand kennt.
- **Deklarative Konfiguration**: Man beschreibt den gewünschten Endzustand des Systems; ein Werkzeug sorgt dafür, dass die Realität dem entspricht – im Gegensatz zu imperativen Befehlen, die den Zustand direkt verändern.
- **Atomare Aktivierung**: Ein Rebuild wird komplett fertiggebaut, bevor überhaupt etwas am laufenden System geändert wird; es gibt keinen halb-aktualisierten Zwischenzustand. Details in Kapitel 2.
- **Nix**: Der Paketmanager und die zugrundeliegende Sprache/Build-Engine, läuft auch ohne NixOS.
- **Nixpkgs**: Die Sammlung der Paketdefinitionen, die Nix baut.
- **Home Manager**: Eigenständiges Projekt für deklarative Nutzer-Konfiguration (Dotfiles), kein Teil von NixOS. Details in Kapitel 13.

## Kapitel 2 – Das Nix-Modell

- **Nix Store** (`/nix/store`): Ablageort für so gut wie alles, was Nix baut oder herunterlädt; Pfade sind unveränderlich und tragen einen 32-stelligen Hash im Namen.
- **Derivation**: Die Bauanleitung für einen Store-Pfad (`.drv`-Datei) – Builder, Argumente, Eingaben, Ausgaben.
- **Input-addressed vs. content-addressed**: Im Standardmodell hängt der Store-Pfad-Hash von den Build-Eingaben ab (input-addressed); das experimentelle Feature "content-addressed derivations" leitet ihn stattdessen vom tatsächlichen Output ab.
- **Profil**: Eine benannte Kette von Generationen (Symlinks), z. B. das Systemprofil unter `/nix/var/nix/profiles/system`.
- **Generation**: Ein nummerierter Symlink (`system-<N>-link`), der auf einen konkreten Store-Pfad zeigt; Grundlage für Rollback.
- **GC-Root**: Ein Ausgangspunkt (z. B. das aktuelle Profil), von dem aus erreichbare Store-Pfade eine Garbage Collection überleben.
- **`/run/current-system`**: Symlink auf den Store-Pfad des aktuell aktiven Systems; `/run/booted-system` zeigt auf das zuletzt gebootete System.
- **Closure**: Die vollständige transitive Hülle aller Abhängigkeiten eines Store-Pfads.

## Kapitel 3 – Nix als Sprache

- **Attribute Set**: Nix' Name/Wert-Struktur, `{ a = 1; b = 2; }`; verschachtelt über Punktnotation als Kurzschreibweise.
- **Currying**: Funktionen mit "mehreren Parametern" sind eigentlich Ketten einparametriger Funktionen, die sich einzeln teilanwenden lassen.
- **`rec`**: Macht ein Attribute Set intern selbstreferenzierend; ohne `rec` sehen sich Geschwister-Attribute nicht.
- **`with`**: Holt alle Namen eines Attribute Sets in den Sichtbereich des folgenden Ausdrucks; bei Namensüberschneidungen unklar, welcher Name gewinnt.
- **`import`**: Eingebaute Funktion, die eine Nix-Datei lädt und auswertet – zu unterscheiden von der modulsystem-eigenen `imports`-Option (Kapitel 5).
- **Lazy Evaluation**: Werte werden nur berechnet, wenn sie tatsächlich gebraucht werden; unbenutzte, fehlerhafte Ausdrücke fallen nicht auf.
- **Pfad-Typ**: Unquotierte Pfadliteralen (`./foo.nix`) sind in Nix ein eigener Werttyp, keine Strings.

## Kapitel 4 – Installation

- **`hardware-configuration.nix`**: Von `nixos-generate-config` erzeugte Datei mit erkannter Hardware/Dateisystemen; wird bei jedem Lauf überschrieben, nicht von Hand pflegen.
- **disko**: Community-Tool für deklarative Partitionierung/Formatierung als Nix-Ausdruck statt manueller `parted`-Befehle.
- **Proxmox-VM vs. Proxmox-LXC**: Zwei grundverschiedene Installationswege – VM bootet wie gewohnt von ISO, LXC braucht stattdessen ein fertiges Container-Template (kein ISO-Boot).
- **`proxmox-lxc.nix`**: Offizielles Nixpkgs-Modul für NixOS-Systeme, die als Proxmox-LXC-Container laufen sollen.
- **nixos-anywhere**: Werkzeug für unbeaufsichtigte Installation über SSH, ohne physischen/virtuellen Konsolenzugriff. Details Kapitel 14.
- **Impermanence**: Ansatz, bei dem das Root-Dateisystem bei jedem Boot verworfen wird und nur explizit markierte Pfade persistieren. Details Kapitel 16.

## Kapitel 5 – Das Modulsystem

- **`options`**: Der deklarative Teil eines Moduls – welche Einstellungen es gibt, mit Typ, Default, Beschreibung.
- **`config`**: Der definierende Teil eines Moduls – welche Werte tatsächlich gesetzt werden.
- **`mkIf`**: Schaltet einen `config`-Block bedingt frei; Standardmuster für Enable-Flags.
- **`mkDefault`/`mkForce`**: Setzen Werte mit niedrigerer bzw. höherer Priorität als eine normale Definition.
- **`mkMerge`**: Führt mehrere `config`-Blöcke zu einem zusammen.
- **`mkEnableOption`**: Kurzform, um eine Standard-Enable-Option (Default `false`) zu deklarieren.
- **`lib`**: Nixpkgs' Standardbibliothek – Modul-Mechanik, Datenverarbeitungsfunktionen, `lib.types`.

## Kapitel 6 – Alltagsbetrieb

- **`nix shell`**: Ad-hoc/temporärer Paketzugriff nur für die aktuelle Shell-Sitzung, nichts wird dauerhaft installiert.
- **`services.openssh`**: Kanonischer Name für den SSH-Dienst; `services.sshd.enable` ist ein Alias nur für das Enable-Flag.
- **`users.mutableUsers`**: Bei `false` sind `/etc/passwd`/`/etc/group` strikt an die Konfiguration gekoppelt, imperative User-Tools funktionieren nicht mehr.
- **`networking.firewall`**: Standardmäßig aktiv; Ports müssen explizit freigegeben werden (`allowedTCPPorts` oder dienstspezifisches `openFirewall`).
- **`systemd.services.<name>`**: Deklarative Definition eigener systemd-Units, inkl. `script` und `serviceConfig`.

## Kapitel 7 – nixos-rebuild im Griff

- **`nixos-rebuild test`**: Baut und aktiviert eine Konfiguration, ohne sie zum Boot-Default zu machen – ein Reboot kehrt zur alten Generation zurück.
- **`nixos-rebuild dry-activate`**: Zeigt, was eine Aktivierung ändern würde, ohne das System tatsächlich anzufassen.
- **`nixos-rebuild switch --rollback`**: Aktiviert die vorherige Generation, ohne neu zu bauen.
- **Build-Fehler vs. Aktivierungsfehler**: Ein Build-Fehler ändert nichts am laufenden System; ein Aktivierungsfehler kann einen Mischzustand hinterlassen, weil die neue Generation schon registriert ist.
- **`specialisation`**: Alternative Boot-Konfiguration als Variante der Hauptkonfiguration, zusätzlich im Bootmenü wählbar.

## Kapitel 8 – Reproduzierbarkeit

- **Channel**: Rollender, nicht versionierter Zeiger auf einen Nixpkgs-Stand, verwaltet über `nix-channel`.
- **`flake.nix`**: Deklariert `inputs` (Abhängigkeiten mit URL) und `outputs` (u. a. `nixosConfigurations`); `outputs` ist eine gewöhnliche Nix-Funktion.
- **`flake.lock`**: JSON-Datei, die jeden Input auf exakten Commit- (`rev`) und Inhalts-Hash (`narHash`) fixiert.
- **`follows`**: Erzwingt, dass ein verschachtelter Input (z. B. Home Managers eigenes nixpkgs) demselben Input wie das Hauptsystem folgt.
- **`nix flake update`**: Aktualisiert alle Inputs; mit Inputname als Argument gezielt nur einen.

## Kapitel 9 – Updates & Wartung

- **Switch Inhibitors**: Seit 26.05 eingebaute Prüfung, die den Wechsel zu einer neuen Generation verweigert, wenn sich kritische Werte (z. B. systemd-Version) geändert haben, außer bei `NIXOS_NO_CHECK=1`.
- **`boot.initrd.systemd.enable`**: Schaltet den seit 26.05 standardmäßigen systemd-basierten Stage 1 ab (Rückfall auf die deprecated gescriptete Variante, laut Release Notes nicht empfohlen).
- **`nix.gc`**: NixOS-Optionen für automatisierte Garbage Collection über einen systemd-Timer.
- **`configurationLimit`**: Begrenzt die Zahl der im Bootloader behaltenen Generationen – unabhängig von allgemeiner Garbage Collection.
- **Store-Optimierung**: Ersetzt inhaltsgleiche Dateien im Store durch Hardlinks (`nix-store --optimise` bzw. `auto-optimise-store`).

## Kapitel 10 – Secrets

- **sops-nix**: Verschlüsselt Secrets-Dateien mit GPG oder age; entschlüsselt zur Laufzeit unter `/run/secrets/…`.
- **agenix**: Schlankere Alternative, ausschließlich age, oft direkt mit SSH-Host-Keys; entschlüsselt unter `/run/agenix/…`.
- **`hashedPasswordFile`**: Verweist auf einen Laufzeit-Pfad statt den Passwort-Hash direkt (und damit im Store) zu speichern.
- **`builtins.readFile` auf Secret-Pfaden**: Kopiert das entschlüsselte Secret ohne Fehlermeldung in den Store – der gefährlichste Secrets-Fehler, weil er unsichtbar bleibt.

## Kapitel 11 – Fehlersuche

- **Evaluation- vs. Build-Fehler**: Evaluation-Fehler (vor jedem Build, brauchen `--show-trace`) vs. Build-Fehler (beim eigentlichen Bauen, brauchen `nix log`).
- **`nix repl`**: Interaktive Umgebung zum schrittweisen Aufbauen und Inspizieren von Nix-Ausdrücken.
- **`nix eval`**: Liefert den Wert einer einzelnen, bereits zusammengeführten Option, ohne vollen Rebuild.
- **`journalctl -b -1`**: Zeigt das Log des vorherigen (nicht des aktuellen) Boots – wichtig nach einem Absturz.
- **Infinite recursion**: Eine Option hängt direkt oder indirekt von ihrem eigenen Wert ab.
- **`--keep-failed`**: Lässt das temporäre Verzeichnis eines fehlgeschlagenen Builds zur manuellen Untersuchung stehen.
- **Lix / Determinate Nix**: Alternative Nix-Implementierungen – Lix ein Community-Fork, Determinate Nix ein kommerzieller Downstream (kein Fork).

## Kapitel 12 – Software finden & anpassen

- **Overlay**: Funktion `final: prev: { … }`, die das gemeinsame `pkgs`-Set global verändert, eingebunden über `nixpkgs.overlays`.
- **`override`**: Ändert die Argumente, mit denen eine paketerzeugende Funktion aufgerufen wurde.
- **`overrideAttrs`**: Ändert direkt das an `stdenv.mkDerivation` übergebene Attribut-Set (Version, Quelltext, Patches, …).
- **`allowUnfree`/`permittedInsecurePackages`**: Schalten standardmäßig blockierte unfreie bzw. unsichere Pakete gezielt frei.
- **`stdenv.mkDerivation`**: Grundbaustein für eigene Pakete; alles, was andere Pakete nutzen sollen, muss unter `$out` landen.

## Kapitel 13 – Home Manager

- **Home Manager Standalone**: `home-manager switch`, unabhängig vom System-Rebuild; auf Nicht-NixOS-Systemen einziger Weg.
- **Home Manager NixOS-Modul**: `home-manager.users.<name>` in der Systemkonfiguration, wird mit `nixos-rebuild switch` mitgebaut.
- **`home.stateVersion`**: Pflichtoption ohne Default, markiert Home-Manager-Kompatibilität; nach Ersteinrichtung nicht ohne Release-Notes-Check ändern.
- **`useGlobalPkgs`**: Lässt Home Manager dasselbe `pkgs`-Set wie das System verwenden statt ein eigenes zu instanziieren.

## Kapitel 14 – Mehrere Maschinen

- **`nixos-rebuild --target-host`**: Eingebauter Remote-Deploy-Weg ohne Zusatztools, ohne eingebaute Parallelisierung.
- **deploy-rs**: Mehrere Profile pro Node; automatischer "Magic Rollback", wenn die Zielmaschine sich nach der Aktivierung nicht als gesund zurückmeldet.
- **colmena**: Paralleles, tag-basiertes Deployment vieler Hosts; `apply` ohne `--on` trifft standardmäßig alle Hosts.

## Kapitel 15 – Desktop (kompakt)

- **`services.displayManager.*`/`services.desktopManager.*`**: Seit 25.11 die korrekten, obersten Optionspfade für Display-/Desktop-Manager – nicht mehr unter `services.xserver` verschachtelt.
- **`hardware.graphics.enable`**: Nachfolger von `hardware.opengl.enable`, aktiviert den Grafik-Unterbau (OpenGL/Vulkan).
- **NVIDIA-Sonderfall**: Proprietärer Treiber mit eigener Optionsgruppe `hardware.nvidia.*`, historisch häufigste Quelle für Wayland-Desktop-Bugs.

## Kapitel 16 – Disaster Recovery

- **"Erase your darlings"**: Ursprungskonzept (Graham Christensen) – Root-Dateisystem bei jedem Boot auf einen leeren Zustand zurücksetzen.
- **`environment.persistence`**: Option des impermanence-Moduls, listet Verzeichnisse/Dateien, die einen Reboot überleben sollen.
- **tmpfs-Root vs. ZFS-/Btrfs-Rollback**: Zwei Umsetzungswege für Impermanence – RAM-basiert (einfach, stromausfallanfällig) vs. Snapshot-Rollback (robuster).
- **Was NixOS absichert vs. Nutzdaten**: Systemkonfiguration ist aus dem Repo reproduzierbar, Nutzdaten (Datenbanken, Uploads, Logs) brauchen eine eigene Backup-Strategie.
