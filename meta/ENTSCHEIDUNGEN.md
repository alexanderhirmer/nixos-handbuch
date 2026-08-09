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
- `--ostype unmanaged` und der Nixpkgs-Default `proxmoxLXC.manageNetwork
  = false` treffen gegenläufige Annahmen und ergeben zusammen einen
  Container **ohne IP und ohne DNS**: Proxmox' Setup-Plugin
  `PVE::LXC::Setup::Unmanaged` hat leere `setup_network`/`set_hostname`/
  `set_dns`-Rümpfe, während `proxmox-lxc.nix` bei `manageNetwork = false`
  genau `useDHCP = false; useNetworkd = true; useHostResolvConf = false;`
  setzt und darauf wartet, dass Proxmox etwas hinterlegt hat. Deshalb
  trägt schon `bootstrap.nix` (Schritt 1) `proxmoxLXC.manageNetwork =
  true;` und `networking.useDHCP = true;` — sonst kann Schritt 2 von
  innen kein `nixpkgs` laden. Beim Übertragen der Baseline auf Projekt 2/3
  (VMs statt LXC) entfällt dieser Sonderfall.
- `pve.proxmox.com` und `git.proxmox.com` sind aus der Arbeitsumgebung
  nicht erreichbar. Belastbarer Ersatz für Proxmox-Behauptungen ist der
  GitHub-Spiegel des Quellcodes (`github.com/proxmox/pve-container`),
  nicht Community-Gists — so sind `pct push`/`pct pull` belegt.
- Ob `/etc/services` unter NixOS nach einer SSH-Port-Änderung angepasst
  werden muss: **Noch nicht abschließend im Schritt dokumentiert** —
  offener Recherchepunkt für den SSH-Härtung-Schritt (voraussichtlich
  nein, da NixOS-Dienste den Port direkt aus der jeweiligen
  Modul-Option beziehen, nicht per Name-Lookup über `/etc/services`,
  und `/etc/services` unter NixOS ohnehin generiert statt von Hand
  gepflegt wird — das ist eine vorläufige Einschätzung, muss im
  entsprechenden Schritt noch sauber mit Quelle belegt werden).

## Projekt 1 — Dateikontrakt (verbindlich ab Schritt 2)

Damit die Schritte 2–16 zusammenpassen (und Projekt 2/3 die Baseline
übernehmen können), steht die Repo-Struktur des Lesers vorab fest.
Kein Schritt erfindet eigene Pfade oder Modulnamen.

```
<repo-root>/
├── flake.nix                        Schritt 2
├── flake.lock                       Schritt 2 (erzeugt, gepinnt)
├── hosts/
│   └── <hostname>/
│       ├── bootstrap.nix            Schritt 1 (nur fürs Template)
│       ├── create-container.sh      Schritt 1
│       └── configuration.nix        Schritt 2 (Host-Einstieg)
└── modules/
    ├── baseline/
    │   ├── default.nix              Schritt 3 (Sammel-Import, wächst per Diff)
    │   ├── users.nix                Schritt 3
    │   ├── sudo.nix                 Schritt 4
    │   ├── ssh.nix                  Schritt 5 (+ Klärung Schritt 6)
    │   ├── fail2ban.nix             Schritt 7
    │   ├── ldap.nix                 Schritt 8
    │   └── firewall.nix             Schritt 9
    └── runner/
        ├── default.nix              Schritt 10 (Sammel-Import)
        ├── container-runtime.nix    Schritt 10
        ├── forgejo-runner.nix       Schritt 11
        └── runner.env               Schritt 12 (Klartext-Variante)
```

Regeln dazu:

- `hosts/<hostname>/configuration.nix` importiert `../../modules/baseline`
  und (ab Schritt 10) `../../modules/runner`; die beiden `default.nix`
  sammeln ihre Geschwisterdateien in `imports`. Jeder Schritt ab 4 legt
  **eine** neue Moduldatei an und ergänzt genau eine Zeile in der
  passenden `default.nix` (Diff-Stil).
- Im Container ist dasselbe Repo unter `/etc/nixos` ausgecheckt.
  Rebuild-Kommando durchgängig:
  `nixos-rebuild switch --flake /etc/nixos#<hostname>`
  (bzw. `build` statt `switch` zum Testen).
- Instanzname des Runners in
  `services.gitea-actions-runner.instances.<runner-name>` ist der
  Platzhalter `<runner-name>` aus der Tabelle.
- Der sops-age-Exkurs (Schritt 15) ersetzt `modules/runner/runner.env`
  nicht, sondern stellt ihm `secrets/runner.env` (verschlüsselt) plus
  `.sops.yaml` unter `<repo-root>/` gegenüber.
- Dateinamen der Buchkapitel = Schrittnummer: `NN-<slug>.md` unter
  `projekt-1-forgejo-runner/`, `weight: NN`. Schritt 6 bleibt ein
  eigener (kurzer) Schritt, damit Datei- und Schrittnummern 1:1 bleiben.

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
