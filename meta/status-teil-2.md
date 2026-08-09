# Status Teil II

Freigegebene Schrittpläne aller drei Projekte (Phase 2, vom Nutzer mit
"Das passt" bestätigt). `[x]` = geschrieben und im Repo, `[ ]` = offen.
Jede Zeile ist der verbindliche Ergebnissatz für den jeweiligen Schritt
— beim Schreiben nicht neu erfinden, nur ggf. technisch verfeinern.

## Projekt 1: Gehärteter Forgejo-Runner-LXC

Ordner: `projekt-1-forgejo-runner/`

- [x] 0. Architekturüberblick und Platzhaltertabelle stehen (Hostname, IP, LDAP-Domain, Forgejo-URL, Admin-User-Schema). → `00-uebersicht.md`
- [x] 1. Der LXC-Container ist über ein versioniertes Skript (feste `pct create`-Parameter) angelegt, nicht per Ad-hoc-Eingabe. → `01-container-bootstrap.md`
- [ ] 2. Ein Flake-Grundgerüst mit `configuration.nix` und gesetztem `hostName` baut erfolgreich (per `nixos-rebuild build` von innen getestet, dann `switch`).
- [ ] 3. Der lokale Nutzer `f-local-admin-<hostname>` existiert, ausschließlich mit hinterlegtem SSH-Key, kein Passwort gesetzt.
- [ ] 4. `security.sudo.extraRules` gewährt ausschließlich diesem Nutzer passwortloses sudo.
- [ ] 5. SSH läuft auf Port 40, `PermitRootLogin` ist deaktiviert. (Inkl. Klärung `/etc/services`, siehe `ENTSCHEIDUNGEN.md`.)
- [ ] 6. Geklärt und dokumentiert ist, ob `/etc/services` unter NixOS überhaupt manuell angepasst werden muss. (Kann mit Schritt 5 zusammengelegt werden, wenn sich das beim Schreiben anbietet.)
- [ ] 7. `fail2ban` ist aktiv und schützt den SSH-Dienst auf dem neuen Port.
- [ ] 8. `sssd` bindet den Container an das bestehende LDAP an, ein LDAP-Testnutzer kann sich per Passwort einloggen.
- [ ] 9. Die Firewall lässt ausschließlich Port 40 (und was der Runner sonst braucht) herein.
- [ ] 10. `virtualisation.podman` (oder docker) ist aktiv, damit der Runner Container-Jobs ausführen kann.
- [ ] 11. `services.gitea-actions-runner` ist mit `package = pkgs.forgejo-runner` konfiguriert.
- [ ] 12. Eine `.env`-Datei mit Klartext-Werten ist angelegt und vom Runner eingebunden.
- [ ] 13. Der Runner ist mit einem in der Forgejo-Weboberfläche erzeugten Token registriert und online.
- [ ] 14. Ein Test-Workflow läuft erfolgreich über den neuen Runner durch.
- [ ] 15. Exkurs: dieselbe `.env` liegt stattdessen sops-age-verschlüsselt vor, der Unterschied zur Klartext-Variante ist erklärt.
- [ ] 16. Projektabschluss: vollständiger Repo-Baum, gesammelte Endkonfiguration, 3–5 Ausbaustufen, Teardown-Anleitung stehen.

**Zuletzt erreichter Zustand:** Container `<vmid>` läuft, minimal per `pct enter` erreichbar, noch keine NixOS-Konfiguration über das Bootstrap-Modul hinaus angewendet.

## Projekt 2: Vaultwarden mit Reverse Proxy, Backup & Restore

Ordner: `projekt-2-vaultwarden/` (noch nicht angelegt)

- [ ] 0. Architekturüberblick und Platzhaltertabelle stehen (App-Host, DB-Host, interne Domain).
- [ ] 1. Baseline-Härtung (local-admin, sudo, SSH-Port+fail2ban, LDAP) ist als gemeinsames Modul übernommen — Unterschied zu Projekt 1: VM statt LXC, disko statt `pct create`.
- [ ] 2. Beide VMs sind über eine disko-Konfiguration deklarativ partitioniert und installiert.
- [ ] 3. Ein Flake mit zwei `nixosConfigurations` und einem gemeinsamen Baseline-Modul baut für beide Hosts.
- [ ] 4. Eine private Root-CA signiert die SSH-Host-Zertifikate beider Hosts (Nebenrolle).
- [ ] 5. sops-age ist mit je einem Host-Key eingerichtet, `.sops.yaml` verweist auf beide (Nebenrolle).
- [ ] 6. Postgres läuft auf dem DB-Host, deklarativ konfiguriert.
- [ ] 7. Die DB-Zugangsdaten liegen als sops-Secret vor, nicht im Klartext.
- [ ] 8. Vaultwarden läuft auf dem App-Host und verbindet sich über das Secret mit der DB.
- [ ] 9. Der Vaultwarden-Admin-Token liegt ebenfalls als sops-Secret vor.
- [ ] 10. Ein Reverse Proxy terminiert TLS intern (Zertifikat von der eigenen Root-CA) vor Vaultwarden.
- [ ] 11. Die Firewall zwischen den Hosts lässt nur die tatsächlich nötigen Ports zu.
- [ ] 12. Ein minimaler GitOps-Mechanismus wendet Änderungen nach einem Push auf `main` automatisch an (Nebenrolle).
- [ ] 13. Vaultwarden ist im Browser erreichbar, ein Test-Account lässt sich anlegen.
- [ ] 14. Ein Backup-Job sichert DB-Dump und Datenverzeichnis nach Zeitplan.
- [ ] 15. Ein Restore-Test auf einer zweiten Instanz bestätigt, dass das Backup tatsächlich funktioniert.
- [ ] 16. Projektabschluss: Repo-Baum, Endkonfigurationen, Ausbaustufen, Teardown stehen.

**Zuletzt erreichter Zustand:** Noch nicht begonnen.

## Projekt 3: Reproduzierbare Web-Fleet mit Monitoring

Ordner: `projekt-3-fleet/` (noch nicht angelegt)

- [ ] 0. Architekturüberblick und Platzhaltertabelle stehen (3+ Web-/App-Knoten, 1 Monitoring-Knoten).
- [ ] 1. Eine Repo-Struktur mit `hosts/<name>/` und gemeinsamen Modulen ist angelegt.
- [ ] 2. Die Baseline-Härtung und das disko-Schema aus Projekt 1/2 sind als wiederverwendbare Module eingebunden.
- [ ] 3. Ein `nixos-anywhere`-Runbook installiert einen neuen Knoten vollautomatisch von der IP bis zum fertigen System.
- [ ] 4. Die ersten zwei Knoten sind darüber erfolgreich bereitgestellt.
- [ ] 5. Ein colmena-Setup mit Tags deckt alle Knoten der Flotte ab.
- [ ] 6. Jeder Web-/App-Knoten läuft mit einem einfachen, gemeinsamen Beispieldienst.
- [ ] 7. Prometheus läuft deklarativ auf dem Monitoring-Knoten.
- [ ] 8. `node-exporter` läuft auf allen Knoten über ein gemeinsames Modul.
- [ ] 9. Grafana zeigt ein Grundgerüst-Dashboard mit Daten aller Knoten.
- [ ] 10. sops-age ist auf mehrere Empfänger-Hosts erweitert — Unterschied zu Projekt 2: mehrere statt zwei Ziele.
- [ ] 11. Eine CI-Pipeline baut jede Host-Konfiguration vor einem Merge und bricht bei Fehlern ab.
- [ ] 12. Nach einem Merge auf `main` rollt dieselbe Pipeline die Änderung automatisch auf die Flotte aus.
- [ ] 13. Ein weiterer Knoten ist allein durch Repo-Eintrag plus `nixos-anywhere`-Lauf hinzugefügt, als Reproduzierbarkeits-Nachweis.
- [ ] 14. Ein Knoten wird bewusst zerstört.
- [ ] 15. Der Knoten ist ausschließlich aus Repo und Backup wiederhergestellt, die nötigen Schritte und die Dauer sind dokumentiert.
- [ ] 16. Projektabschluss: Repo-Baum, Endkonfigurationen, Ausbaustufen, Teardown stehen.

**Zuletzt erreichter Zustand:** Noch nicht begonnen.

## Nächster Schritt insgesamt

Projekt 1, Schritt 2 (Flake-Grundgerüst + `nixos-rebuild` von innen).
