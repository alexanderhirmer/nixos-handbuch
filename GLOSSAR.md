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

## Teil II, Projekt 1 – Gehärteter Forgejo-Runner-LXC

### Schritt 1 (Container anlegen)

- **`--ostype unmanaged`**: `pct create`-Flag für Betriebssystemtypen, die Proxmox nicht kennt (wie NixOS). Proxmox weist dem Container dafür das Setup-Plugin `PVE::LXC::Setup::Unmanaged` zu, dessen Methoden `setup_network`, `set_hostname` und `set_dns` leere Rümpfe sind – Proxmox schreibt also nichts in den Gast, obwohl z. B. `ip=dhcp` gesetzt ist.
- **`proxmoxLXC.manageNetwork`**: Option aus `proxmox-lxc.nix`. Default `false` lässt das Modul `useDHCP = false; useNetworkd = true; useHostResolvConf = false;` setzen – es erwartet dann fertige Netzwerkdaten von Proxmox. Weil `--ostype unmanaged` diese Daten nie liefert, wird die Option in diesem Projekt schon im Bootstrap-Template auf `true` gesetzt; `networking.useDHCP = true` übernimmt die Adressvergabe stattdessen selbst.

### Schritt 2 (Flake-Grundgerüst)

- **`proxmoxLXC.manageHostName`**: Ebenfalls aus `proxmox-lxc.nix`, analog zu `manageNetwork` (Schritt 1): Bei `false` erzwingt das Modul `hostName = mkForce ""` und erwartet den Hostnamen von Proxmox. Hier auf `true` gesetzt, weil `--ostype unmanaged` ihn nie liefert.
- **`pct push`/`pct pull`**: Proxmox-CLI-Befehle, die genau eine Datei zwischen Proxmox-Host und einem laufenden LXC-Container kopieren – push hinein, pull heraus. Funktionieren unabhängig vom Netzwerkzustand des Gasts; laut Proxmox-Quellcode setzt `push` einen laufenden Container voraus.
- **Bootstrap-Aktivierung von Flakes (`/etc/nix/nix.conf`)**: Vor dem ersten erfolgreichen `switch` kann `nix.settings.experimental-features` aus der eigenen Konfiguration noch nicht greifen; ein einmaliges `echo "experimental-features = nix-command flakes" >> /etc/nix/nix.conf` schaltet Flakes für die ersten Aufrufe von außen frei.

### Schritt 3 (Lokaler Admin-Nutzer)

- **`hashedPassword = null`**: NixOS-Default für `users.users.<name>.hashedPassword` – laut Options-Beschreibung "this user will not be able to log in using a password". In Kombination mit `users.mutableUsers = false` schreibt die Aktivierung ein literales `!` ins Passwortfeld von `/etc/shadow`.
- **`allowsLogin`-Semantik (`/etc/shadow`-Feld)**: `""` (leer) erlaubt laut Nixpkgs-Quellcode Login *ohne* Passwort; `!`, `!!`, `*` oder `null` verhindern Passwort-Login vollständig; jeder andere Wert gilt als echter Hash.
- **Lockout-Schutzassertion**: NixOS bricht den Build bei `users.mutableUsers = false` ab, wenn weder `root` noch ein `wheel`-Mitglied ein funktionierendes Login (Passwort oder SSH-Key) hätte.
- **`update-users-groups.pl`**: Das Perl-Skript, das `/etc/passwd`, `/etc/group` und `/etc/shadow` aus der deklarierten Nutzerkonfiguration erzeugt (Standardpfad, solange weder `systemd.sysusers` noch `services.userborn` aktiv sind).

### Schritt 4 (Passwortloses Sudo)

- **`security.sudo.extraRules`**: Listenwertige NixOS-Option für zusätzliche `sudoers`-Regeln; das Modul selbst trägt darüber bereits Default-Regeln für `root` und `wheel` ein. Bei mehreren passenden Zeilen für denselben Nutzer gewinnt in `/etc/sudoers` die letzte – die Reihenfolge in der zusammengeführten Liste entscheidet also über die Wirkung.
- **`security.sudo.wheelNeedsPassword`**: Schaltet Passwortpflicht für die gesamte `wheel`-Gruppe ab, wenn `false` – NixOS-Default ist `true`.
- **`lib.mkOrder`**: Die allgemeine Form hinter `mkBefore`/`mkAfter` (Kapitel 5) – legt die Position eines Werts in einer per `mkMerge` zusammengeführten Liste fest, unabhängig davon, welcher *Wert* bei einem echten Konflikt gewinnt (das regeln `mkDefault`/`mkForce`).

### Schritt 5 (SSH-Härtung)

- **`services.openssh.settings`**: Freeform-Submodule für `sshd_config`-Direktiven; wandelt Nix-Attribute automatisch in gültige `sshd_config`-Syntax um (u. a. `true`/`false` → `yes`/`no`).
- **`services.openssh.extraConfig`**: Verbatim-Text, der ans Ende der generierten `sshd_config` angehängt wird – der einzig syntaktisch gültige Ort für `Match`-Blöcke, weil er immer hinter den aus `settings` erzeugten Zeilen liegt. In diesem Projekt bewusst nicht für einen `Match`-Block genutzt.
- **`Type = "notify-reload"` (systemd)**: Service-Typ, bei dem eine geänderte Unit nicht hart neu gestartet, sondern per Reload-Mechanismus aktualisiert wird – bestehende Verbindungen/Sitzungen bleiben dabei unangetastet. `sshd` nutzt diesen Typ.

### Schritt 6 (Exkurs: /etc/services)

- **`pkgs.iana-etc`**: Nixpkgs-Paket, das ein vorgefertigtes Release-Tarball des Projekts `Mic92/iana-etc` (Republishing der IANA-Port-/Protokoll-Registry) lädt und als `/etc/services`/`/etc/protocols` bereitstellt – keine Eigenkompilierung, keine Kenntnis der tatsächlichen Dienstkonfiguration.
- **`environment.etc.<name>.source` vs. `.text`**: Zwei sich gegenseitig ausschließende Wege, den Inhalt einer generierten `/etc`-Datei zu bestimmen; `/etc` wird bei jeder Aktivierung komplett neu aus `environment.etc` aufgebaut.

### Schritt 7 (fail2ban)

- **fail2ban-„Jail"**: Kombination aus Filter (Logauswertung) und Action (z. B. IP sperren), pro Dienst konfigurierbar; die spezielle `DEFAULT`-Jail liefert Fallback-Werte für alle anderen Jails. NixOS bringt ein vorkonfiguriertes `sshd`-Jail mit, das seinen Port automatisch aus `services.openssh.ports` übernimmt.
- **`backend = "systemd"` (fail2ban)**: Ein Jail liest Fehlversuche direkt aus dem systemd-Journal statt eine Logdatei zu parsen.
- **`bantime-increment`**: fail2ban-Mechanismus, der die Sperrdauer bei wiederholten Verstößen derselben IP progressiv verlängert (Standardformel: Faktor 1, 2, 4, 8, 16 … × `bantime`).

### Schritt 8 (LDAP-Anbindung mit sssd)

- **sssd (System Security Services Daemon)**: Dienst, der einen oder mehrere Identitäts-/Auth-Provider (z. B. LDAP) bündelt und dem restlichen System einheitlich über NSS und PAM zur Verfügung stellt.
- **NSS (Name Service Switch)**: Der über `/etc/nsswitch.conf` gesteuerte Umschalter, welche Quellen (`files`, `sss`, …) für Lookups wie `passwd`/`group`/`shadow` befragt werden; unter NixOS aus `system.nssDatabases` generiert.
- **pam_sss**: PAM-Modul aus sssd, das Authentifizierung sowie Account-/Session-Handling gegen die konfigurierten sssd-Domains übernimmt; wird von NixOS automatisch in jeden Standard-PAM-Service eingehängt, sobald `services.sssd.enable = true`.
- **pam_mkhomedir**: PAM-Session-Modul, das beim Login ein fehlendes `$HOME` anlegt – unter NixOS aktiviert über `security.pam.services.<name>.makeHomeDir`.
- **Bind-DN (Default Bind)**: Distinguished Name, mit dem sssd sich gegenüber LDAP authentisiert, um Nutzer zu *suchen* (`ldap_default_bind_dn`/`ldap_default_authtok`) – zu unterscheiden vom Login-Bind, bei dem sssd sich testweise mit den Zugangsdaten des einloggenden Nutzers verbindet.

### Schritt 9 (Firewall)

- **nftables**: Linux-Firewall-Framework, Nachfolger von iptables/xtables; verwaltet Regeln über den Kernel-Mechanismus `nf_tables`.
- **iptables-nft (`xtables-nft-multi`)**: Kompatibilitätsschicht, die die klassische `iptables`-Kommandozeile beibehält, Regeln intern aber in `nf_tables` übersetzt – unter NixOS der Default, weil `pkgs.iptables` mit `nftablesCompat = true` gebaut wird.
- **CAP_NET_ADMIN**: Linux-Capability für Netzwerkkonfiguration und Netfilter/nftables-Verwaltung; in einem unprivilegierten LXC-Container innerhalb des eigenen Namespace vorhanden, ohne dass Proxmox' `nesting`-Feature dafür nötig wäre.
- **Proxmox-Firewall**: Von der Gast-Firewall unabhängige, zusätzliche Filterschicht auf dem Proxmox-Host; pro Container über `/etc/pve/firewall/<vmid>.fw` konfiguriert, pro Netzwerkinterface über den Parameter `firewall=<0|1>` scharf geschaltet.

### Schritt 10 (Container-Laufzeit: Podman)

- **`virtualisation.podman`**: NixOS-Modul für Podman, einen daemonlosen, Docker-API-kompatiblen Container-Manager; `enable = true` aktiviert u. a. einen systemd-Socket, der den Daemon-Prozess erst bei Bedarf startet.
- **Nesting (LXC)**: Proxmox-Container-Feature (`pct set --features nesting=1`), das einem LXC-Container erlaubt, selbst wieder Container-/Mount-Namespaces aufzuspannen – Voraussetzung für jede Container-Laufzeit innerhalb eines LXC-Gasts.
- **`keyctl`-Feature (LXC)**: Proxmox-Container-Feature, das in unprivilegierten Containern den sonst per Seccomp blockierten `keyctl()`-Syscall freigibt.
- **fuse-overlayfs**: Nutzerraum-Implementierung von OverlayFS auf Basis von FUSE; springt ein, wo native Kernel-Overlay-Mounts mangels `CAP_SYS_ADMIN` (typisch in unprivilegierten/genesteten Containern) scheitern könnten.
- **`mount_program` (containers/storage)**: Storage-Option, die Podman/Docker anweist, Overlay-Mounts über ein externes Programm (z. B. `fuse-overlayfs`) statt über den Kernel direkt vorzunehmen.

### Schritt 11 (Forgejo-Runner-Dienst)

- **`services.gitea-actions-runner`**: NixOS-Modul für den Gitea-/Forgejo-Actions-Runner; `package` ist eine globale Option (Default `pkgs.gitea-actions-runner`, hier auf `pkgs.forgejo-runner` gesetzt), jede konkrete Registrierung liegt unter `instances.<name>` (Submodule mit `enable`, `name`, `url`, `token`/`tokenFile`, `labels`, `settings`, `hostPackages`).
- **Label-Syntax (Actions-Runner)**: `<label>:docker://<image>` bindet ein `runs-on:`-Label an ein Container-Image (vor jedem Job gezogen); `<label>:host` führt den Job stattdessen direkt auf dem Runner-Host aus, ohne Container. Eine leere `labels`-Liste bedeutet die Upstream-Default-Labels, die zwingend Docker/Podman voraussetzen.
- **`escapeSystemdPath`**: Nixpkgs-Hilfsfunktion (`nixos/lib/utils.nix`), die einen String wie einen Dateipfad behandelt und dabei u. a. jeden literalen Bindestrich zu `\x2d` escaped – erzeugt reale systemd-Unit-Namen, die von der geschriebenen Nix-Konfiguration optisch abweichen (`ci-runner-01` → `ci\x2drunner\x2d01`). Cross-Check-Tool: `systemd-escape`.

### Schritt 12 (Runner-Konfiguration als .env)

- **`unitOption` (Merge-Typ)**: Der Nixpkgs-Options-Typ hinter `systemd.services.<name>.serviceConfig.*`; merged mehrere Definitionen automatisch zu einer Liste, sobald mindestens eine der Definitionen selbst eine Liste ist – sonst gilt Gleichheits-Zwang (`mergeEqualOption`).
- **`utils`-Modulargument**: Ein von NixOS an jedes Modul optional gereichtes Spezial-Argument (neben `config`, `lib`, `pkgs`) mit Hilfsfunktionen wie `escapeSystemdPath` – muss im Funktionskopf (`{ pkgs, utils, ... }:`) explizit benannt werden, um im Modulkörper nutzbar zu sein.

### Schritt 13 (Runner registrieren)

- **Registrierungs-Token**: Einmalig in der Forgejo-Weboberfläche erzeugtes Token, mit dem sich ein `forgejo-runner`-Prozess bei einer Forgejo-Instanz anmeldet. Wiederverwendbar für mehrere Runner-Prozesse, kein Single-Use-Token.
- **`tokenFile` vs. `token`** (`services.gitea-actions-runner.instances.<name>`): `token` landet wortwörtlich in der generierten, weltlesbaren Store-Unit; `tokenFile` verweist stattdessen auf einen externen Pfad, der als `EnvironmentFile=` dient und vom systemd-Manager (root) noch vor jedem Privilegien-Wechsel gelesen wird. Eine Assertion erzwingt genau eines von beiden.
- **`.runner`-Datei**: Marker-/Statusdatei unter `/var/lib/gitea-runner/<name>/`, die eine erfolgreiche Registrierung festhält (inkl. eines von der Registrierung verschiedenen Laufzeit-Credentials). Fehlt sie, registriert sich der Runner beim nächsten Start neu.

### Schritt 14 (Test-Workflow)

- **`.forgejo/workflows/`**: Vorrangiges Verzeichnis für Forgejo-Actions-Workflows; fehlt es, fällt Forgejo auf `.github/workflows/` zurück. Sind beide vorhanden, führt Forgejo (anders als GitHub) Workflows aus beiden Verzeichnissen aus.
- **`runs-on`-Label-Matching**: Der Teil eines registrierten Runner-Labels vor dem ersten Doppelpunkt (z. B. `ubuntu-latest` in `ubuntu-latest:docker://node:20-bookworm`) ist der Name, den `runs-on:` referenziert – der Teil danach legt intern fest, wie der Job ausgeführt wird, ist für den Workflow-Autor aber unsichtbar.
- **`DEFAULT_ACTIONS_URL`**: `app.ini`-Option (`[actions]`) einer Forgejo-Instanz, die bestimmt, von wo unqualifizierte `uses:`-Angaben aufgelöst werden.
- **`hostPackages`** (`services.gitea-actions-runner.instances.<name>.hostPackages`): Paketliste, die bei einem `:host`-Schema-Label auf den `$PATH` des Jobs gelegt wird (Default u. a. `bash`, `coreutils`, `curl`, `gawk`, `gitMinimal`, `gnused`, `nodejs`, `wget`) – irrelevant bei `:docker:`-Labels.

### Schritt 15 (Exkurs: sops-age)

- **age-Empfänger (recipient) vs. age-Identität**: Der Empfänger ist der öffentliche Teil (`age1…`) und steht in `.sops.yaml` – für ihn wird verschlüsselt. Die Identität ist der private Teil, mit dem entschlüsselt wird. Ein Host braucht eine Identität, ein Repo kennt nur Empfänger. Die Trennung entscheidet darüber, ob ein privater Schlüssel auf einem Zielsystem liegen muss.
- **`ssh-to-age`**: Tool, das einen vorhandenen SSH-Key (Host- oder Nutzer-Key, Ed25519) in einen age-kompatiblen Schlüssel umrechnet. Erzeugt kein neues Schlüsselmaterial, sondern rechnet vorhandenes um – so bekommt ein Host eine eigene age-Identität, ohne dass ein privater Schlüssel dorthin kopiert werden muss.
- **`sops.age.sshKeyPaths`**: sops-nix-Option, die angibt, welche SSH-Host-Keys als age-Identität zur Entschlüsselung dienen; Default ist bereits die Liste der ed25519-Keys aus `config.services.openssh.hostKeys`. Wer stattdessen ausschließlich einen mitgebrachten Schlüssel nutzen will, muss sie ausdrücklich auf `[ ]` setzen – sonst hängt sops-nix den Host-Key zusätzlich ein.
- **`sops.age.keyFile`**: Pfad zu einer bereits vorhandenen privaten age-Schlüsseldatei auf dem Zielsystem. Typisiert als `pathNotInStore`, siehe dort.
- **`sops.age.generateKey`**: Schaltet das Erzeugen eines age-Schlüssels ein, falls unter `keyFile` keiner liegt. Default ist `false` – "the key must already be present at the specified location". Für Setups, die einen bestehenden Schlüssel weiterverwenden, ist der Default bereits der richtige Wert.
- **`pathNotInStore`** (Nixpkgs-`lib.types`): Pfad-Typ, der Werte unterhalb von `/nix/store` zur Auswertungszeit ablehnt. Wird für Optionen verwendet, die auf Geheimnisse zeigen – er macht das "nicht in den Store kopieren" vom guten Vorsatz zur erzwungenen Regel.
- **`sops updatekeys`**: sops-Unterbefehl, der eine bereits verschlüsselte Datei an die aktuellen `creation_rules` der `.sops.yaml` anpasst. Nötig, sobald ein Empfänger dazukommt oder wegfällt – ohne diesen Lauf kann ein neu eingetragener Empfänger bestehende Dateien nicht entschlüsseln.
- **`format = "dotenv"`** (sops-nix `sops.secrets.<name>.format`): Behandelt die referenzierte sops-Datei als `.env`-Datei; anders als bei YAML/JSON wird dabei nie ein einzelner Schlüssel extrahiert, sondern immer die gesamte entschlüsselte Datei als ein Secret ausgegeben (wie bei `binary`/`ini`). Die separate Option `sops.secrets.<name>.key` ist für dieses Format laut sops-nix-Quellcode wirkungslos.
- **Store-Pfad-Literal vs. String-Pfad (bei Secrets)**: Ein Nix-Ausdruck wie `./runner.env` (Pfad-Literal, Kapitel 3) wird beim Bauen automatisch und weltlesbar in den Store kopiert; ein gleichlautender String wie `config.sops.secrets.x.path` bleibt reiner Text und verweist erst zur Aktivierungszeit auf einen Pfad außerhalb des Stores (bei sops-nix: `/run/secrets/…`, tmpfs).
