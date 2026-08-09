# Entscheidungen & Konventionen

Alles hier ist entweder eine Vorgabe des Nutzers oder eine begründete
Entscheidung der KI während der Arbeit. Bei Widerspruch zwischen dieser
Datei und einem bereits geschriebenen Kapitel/Schritt gilt: nachfragen,
nicht raten.

## Rahmendaten

- Zielversion Teil I: NixOS 26.05 "Yarara" (verifiziert aktuell, Stand
  August 2026; Nachfolger 26.11 noch nicht erschienen).
- Proxmox-Version Teil II: PVE 9.2, Unterbau Debian 13 "Trixie".
- Sprache: Deutsch, Fachbegriffe/Optionsnamen/CLI-Ausgaben englisch.
- Primärweg Nix: Flakes; Channels werden parallel erklärt, wo relevant.

## Namens- und Stilkonventionen (beide Teile)

- Dateinamen: `NN-slug.md`, zweistellige Nummer = `weight` im
  Frontmatter (`title`, `weight`).
- SSH-Option durchgehend als `services.openssh.*` verwendet, nicht
  `services.sshd.*` (Alias, aber `services.openssh` trägt die
  Unteroptionen wie `settings`, `openFirewall`).
- Boxen: `> 💡 **Nice to know:** …` und `> ⚠️ Ungeprüft: …` /
  `> ⚠️ <Warnung>: …` — Formatierung nicht variieren.
- Quellenangaben pro Kapitel/Schritt als Tabelle "Verwendete
  Befehle/Optionen mit Quelle" (Teil I) bzw. im Schritt selbst über
  Fußnoten/Inline-Links (Teil II).

## Teil-II-Platzhalterkonventionen (projektübergreifend)

- Eigenes Infrastruktur-Repo des Lesers (nicht dieses Buch-Repo):
  `<repo-root>`, Beispielwert `~/nixos-infra`.
- Pro-Host-Verzeichnis darin: `<repo-root>/hosts/<hostname>/`.
- Diese Struktur ist ab Projekt 1 etabliert und wird in Projekt 3 auf
  mehrere Hosts erweitert (nicht neu erfinden).

## Projekt 1 (Forgejo-Runner-LXC) — feststehende Werte

Siehe `projekt-1-forgejo-runner/00-uebersicht.md` für die vollständige
Platzhaltertabelle (`<hostname>`, `<ip>`, `<ldap-uri>`,
`<ldap-base-dn>`, `<forgejo-url>`, `<runner-name>`, `<admin-user>`,
`<pve-storage>`, `<ssh-port>` = 40, `<vmid>` = 201, `<repo-root>`).

Sicherheitsmodell (vom Nutzer korrigiert, verbindlich):
- `f-local-admin-<hostname>`: Login **ausschließlich per SSH-Key**,
  kein Passwort-Login. `sudo` dafür **passwortlos**
  (`security.sudo.extraRules`, nur für diesen einen Nutzer).
- Alle anderen Nutzer (aus LDAP via `sssd`): Login per Passwort,
  **kein** passwortloses `sudo`.
- Passwortloser SSH-Login ist für niemanden sonst erlaubt.
- `fail2ban` ist Teil der Härtung (Nutzer-Vorgabe, nicht vergessen).
- SSH läuft auf Port 40 statt 22.

Recherchierte/verifizierte technische Fakten (Quellen jeweils im
zugehörigen Schritt verlinkt):
- Runner-Modul heißt `services.gitea-actions-runner` (Forgejo nutzt
  denselben Runner wie Gitea); `package = pkgs.forgejo-runner;` wählt
  die Forgejo-Variante des Binaries.
- Eigene, selbst gebaute Proxmox-LXC-Templates werden **nicht** über
  `pveam add` eingespielt (das ist nur für die offiziellen,
  katalogisierten Vorlagen), sondern per Kopie/`scp` direkt nach
  `/var/lib/vz/template/cache/` auf dem jeweiligen Storage.
- `pct create` braucht für NixOS-Container `--ostype unmanaged`, weil
  Proxmox NixOS nicht als bekannten Betriebssystemtyp kennt und sonst
  versuchen würde, Netzwerk/Hostname in (bei NixOS nicht existierende)
  Gastdateien zu schreiben.
- Bootstrap-Strategie: Ein bewusst minimales `proxmox-lxc.nix`-Template
  wird einmalig gebaut und hochgeladen; der Container wird daraus
  angelegt und initial über `pct enter` (kein SSH nötig) erreicht. Die
  eigentliche, vollständige Konfiguration (Hostname, Nutzer, LDAP,
  Härtung, Runner) wird danach **von innen** per `nixos-rebuild`
  angewendet — nicht durch erneutes Bauen/Hochladen des Templates.
  Grund: In Teil I, Kapitel 4 ist dokumentiert, dass `nixos-rebuild`
  innerhalb von Proxmox-LXC-Containern historisch nicht immer
  zuverlässig war; deshalb testet der nächste Schritt zunächst mit
  `nixos-rebuild build`, bevor `switch` läuft.
- Ob `/etc/services` unter NixOS nach einer SSH-Port-Änderung angepasst
  werden muss: **Noch nicht abschließend im Schritt dokumentiert** —
  offener Recherchepunkt für den SSH-Härtung-Schritt (voraussichtlich
  nein, da NixOS-Dienste den Port direkt aus der jeweiligen
  Modul-Option beziehen, nicht per Name-Lookup über `/etc/services`,
  und `/etc/services` unter NixOS ohnehin generiert statt von Hand
  gepflegt wird — das ist eine vorläufige Einschätzung, muss im
  entsprechenden Schritt noch sauber mit Quelle belegt werden).

## Projekt 2 (von der KI vorgeschlagen, freigegeben)

Hauptthema: Vaultwarden (Bitwarden-kompatibler Passwort-Manager) auf
einem App-Host, Postgres auf separatem DB-Host, Reverse Proxy mit
intern signiertem TLS davor, Backup **und geprobter Restore** — bewusst
gewählt, weil Datenverlust bei einem Passwort-Manager besonders teuer
ist. Nebenrollen (Nutzer-Vorgabe): SSH-Host-Zertifikat von privater
Root-CA, Secrets via sops-age, minimales GitOps-Setup.
Baseline-Härtung wird von Projekt 1 übernommen (VM statt LXC, disko
statt `pct create`-Skript — das ist der zu erklärende Unterschied).
Vollständiger, freigegebener Schrittplan: siehe `status-teil-2.md`.

## Projekt 3 (von der KI vorgeschlagen, freigegeben)

Hauptthema: reproduzierbare Fleet aus 3+ Web-/App-Knoten plus einem
Monitoring-Knoten (Prometheus + Grafana — in Teil I nicht behandelt),
komplett aus einem Flake-Repo per `disko` + `nixos-anywhere`
aufsetzbar, Rollout per `colmena`, CI-Gate vor jedem Merge,
abgeschlossen mit einem geprobten Disaster-Recovery-Fall. Baseline und
disko-Schema werden von Projekt 1/2 übernommen. Vollständiger,
freigegebener Schrittplan: siehe `status-teil-2.md`.
