---
title: "Projektabschluss"
weight: 16
---

# Schritt 16: Projekt 1 ist fertig — ein gehärteter Forgejo-Runner-LXC läuft, ist registriert und hat einen grünen Testlauf hinter sich

Die Schritte 1–14 haben aus einem leeren Proxmox-Host einen einzelnen, deklarativ verwalteten LXC-Container gemacht: NixOS-Bootstrap-Template, Baseline-Härtung (lokaler Fallback-Admin, passwortloses Sudo nur für ihn, SSH auf Port `<ssh-port>`, `fail2ban`, LDAP-Login über `sssd`, Firewall) und darauf aufbauend ein Forgejo-Actions-Runner mit Podman-Unterbau, registriert und mit einem echten Workflow-Lauf verifiziert. Dieser Abschluss trägt den Endzustand zusammen: den vollständigen Verzeichnisbaum, jede Datei einmal komplett, die übertragbaren Fähigkeiten, mögliche Ausbaustufen und einen vollständigen Teardown.

## 1. Verzeichnisbaum im Endzustand

```text
<repo-root>/
├── flake.nix                        Schritt 2 angelegt, seither unverändert (Hauptvariante ohne sops-nix)
├── flake.lock                       Schritt 2 (von nixos-rebuild build erzeugt, auf nixos-26.05 gepinnt;
│                                     Inhalt hashbasiert und wird hier nicht nachgebildet)
├── hosts/
│   └── <hostname>/
│       ├── bootstrap.nix            Schritt 1, seither unverändert (nur fürs einmalige Template)
│       ├── create-container.sh      Schritt 1, seither unverändert
│       └── configuration.nix        Schritt 2 angelegt, zuletzt geändert Schritt 10
│                                     (Import von ../../modules/baseline kam in Schritt 3 hinzu,
│                                      von ../../modules/runner in Schritt 10)
└── modules/
    ├── baseline/
    │   ├── default.nix              Schritt 3 angelegt, zuletzt geändert Schritt 9
    │   │                             (Sammel-Import, pro Schritt genau eine neue Zeile)
    │   ├── users.nix                Schritt 3, seither unverändert
    │   ├── sudo.nix                 Schritt 4, seither unverändert
    │   ├── ssh.nix                  Schritt 5, seither unverändert (Schritt 6 war reine Recherche)
    │   ├── fail2ban.nix             Schritt 7, seither unverändert
    │   ├── ldap.nix                 Schritt 8, seither unverändert
    │   └── firewall.nix             Schritt 9, seither unverändert
    └── runner/
        ├── default.nix              Schritt 10 angelegt, zuletzt geändert Schritt 11
        ├── container-runtime.nix    Schritt 10, seither unverändert
        ├── forgejo-runner.nix       Schritt 11 angelegt, geändert Schritt 12 (Endzustand der
        │                             Hauptvariante). Schritt 13 ändert diese Datei NICHT.
        │                             Schritt 15 stellt ihr zwei alternative Fassungen gegenüber
        │                             (Hauptweg und Variante, siehe Exkurs unten), ersetzt sie
        │                             aber nicht.
        └── runner.env                Schritt 12, seither unverändert (Klartext-Variante)
```

Nur relevant, wenn der sops-age-Exkurs (Schritt 15) tatsächlich nachgebaut wird — ergänzt den Baum oben, ersetzt darin nichts:

```text
<repo-root>/
├── .sops.yaml                       Schritt 15 (Exkurs)
└── secrets/
    └── runner.env                   Schritt 15 (Exkurs, sops-age-verschlüsselt)
```

### Was bewusst nicht im Repo liegt

- **`/etc/gitea-runner-<runner-name>-token.env`** (Schritt 13) — das Registrierungstoken aus der Forgejo-Weboberfläche, mit `umask 077` angelegt und auf `root:root`/`600` gesetzt. `forgejo-runner.nix` referenziert den Pfad seit Schritt 11 nur als **String** (`tokenFile`), nicht als Nix-Pfad-Literal — genau deshalb landet der Inhalt nie im weltlesbaren Store.
- **`/var/lib/gitea-runner/<runner-name>/`** — Laufzeitzustand (u. a. die Markerdatei `.runner`), den der Runner-Prozess selbst bei der ersten erfolgreichen Registrierung anlegt (Schritt 13, Prüfkriterium). Kein von Nix verwalteter Pfad; er entsteht und verschwindet mit der Registrierung, nicht mit einem `nixos-rebuild switch`.
- **`/run/secrets/runner-env`** (Schritt 15, Exkurs) — sops-nix legt den entschlüsselten Klartext in ein `tmpfs`, nie in den Store. Existiert ausschließlich, wenn der Exkurs nachgebaut wurde; in der Hauptvariante (Schritt 12) gibt es diesen Pfad nicht, dort liegt `runner.env` stattdessen als Klartext unter `/nix/store/…-runner.env`.
- **`/var/lib/sops-nix/key.txt`** (Schritt 15, Exkurs, nur im Hauptweg) — dein privater age-Schlüssel, per `pct push` von der Workstation auf den Container gebracht, `root:root`/`600`. Weder im Repo noch im Store: `sops.age.keyFile` ist als `pathNotInStore` typisiert, ein Store-Pfad würde vom Modulsystem abgelehnt. In der Variante mit getrennten Rollen existiert diese Datei gar nicht — dort bleibt der private Schlüssel auf der Workstation, und der Container nutzt seinen eigenen SSH-Host-Key als abgeleitete Identität.
- **Die Workflow-Datei aus Schritt 14** (`.forgejo/workflows/runner-test.yaml`) — liegt in einem eigenständigen **Test-Repository auf der Forgejo-Instanz**, nicht in `<repo-root>`. Schritt 14 betont das ausdrücklich: Das NixOS-Infrastruktur-Repo, das diesen Abschluss zusammenfasst, und das Repository, dessen Workflows der Runner ausführt, sind zwei völlig getrennte Orte.

## 2. Gesammelte Endkonfiguration

Platzhalter erscheinen unten genauso wörtlich wie in den Einzelschritten — vor dem Einsatz mit den eigenen Werten aus der Tabelle in `00-uebersicht.md` ersetzen.

### `hosts/<hostname>/bootstrap.nix` (Schritt 1)

```nix
# <repo-root>/hosts/<hostname>/bootstrap.nix
{ modulesPath, ... }:
{
  imports = [ (modulesPath + "/virtualisation/proxmox-lxc.nix") ];

  # Die eine Ausnahme vom Minimalprinzip – Begründung direkt darunter.
  proxmoxLXC.manageNetwork = true;
  networking.useDHCP = true;
}
```

### `hosts/<hostname>/create-container.sh` (Schritt 1)

```bash
#!/usr/bin/env bash
# <repo-root>/hosts/<hostname>/create-container.sh
set -euo pipefail
pct create <vmid> local:vztmpl/nixos-<hostname>-bootstrap.tar.xz \
  --hostname <hostname> \
  --cores 1 \
  --memory 1024 \
  --rootfs <pve-storage>:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --ostype unmanaged \
  --onboot 1
pct start <vmid>
```

### `flake.nix` (Schritt 2, Hauptvariante ohne sops-nix)

```nix
# <repo-root>/flake.nix
{
  description = "Infrastruktur für <hostname> (Forgejo-Runner)";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  };

  outputs = { self, nixpkgs, ... }: {
    nixosConfigurations.<hostname> = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [ ./hosts/<hostname>/configuration.nix ];
    };
  };
}
```

### `hosts/<hostname>/configuration.nix` (Schritt 2 angelegt, Importe aus Schritt 3 und 10 eingearbeitet)

```nix
# <repo-root>/hosts/<hostname>/configuration.nix
{ modulesPath, pkgs, ... }:
{
  imports = [
    (modulesPath + "/virtualisation/proxmox-lxc.nix")
    ../../modules/baseline
    ../../modules/runner
  ];

  # manageNetwork = false (der Default) hiesse: useDHCP = false,
  # useNetworkd = true und hostName = mkForce "" – das Modul erwartet dann
  # Netzwerkdaten und Hostnamen von Proxmox statt aus dieser Datei. Genau
  # das liefert --ostype unmanaged aber nie (siehe Schritt 1). Beide Flags
  # stehen explizit hier, auch wenn manageHostName = true bei aktivem
  # manageNetwork streng genommen schon nichts mehr bewirkt.
  proxmoxLXC = {
    manageNetwork = true;
    manageHostName = true;
  };

  networking = {
    hostName = "<hostname>";
    useDHCP = false;
    interfaces.eth0.ipv4.addresses = [
      { address = "<ip>"; prefixLength = 24; }
    ];
    defaultGateway = "10.20.0.1"; # Beispiel – das Gateway deines Netzes eintragen
  };

  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  environment.systemPackages = [ pkgs.git ];

  system.stateVersion = "26.05";
}
```

### `modules/baseline/default.nix` (Schritt 3 angelegt, wächst bis Schritt 9)

```nix
# <repo-root>/modules/baseline/default.nix
{
  imports = [
    ./users.nix
    ./sudo.nix
    ./ssh.nix
    ./fail2ban.nix
    ./ldap.nix
    ./firewall.nix
  ];
}
```

### `modules/baseline/users.nix` (Schritt 3)

```nix
# <repo-root>/modules/baseline/users.nix
{ ... }:
{
  # Ab hier ist jede Kontoänderung ausschließlich über diese Datei erlaubt –
  # kein `useradd`, kein `passwd` auf der laufenden Maschine.
  users.mutableUsers = false;

  users.users."<admin-user>" = {
    isNormalUser = true;
    description = "Lokaler Fallback-Admin (kein LDAP)";
    extraGroups = [ "wheel" ]; # nötig für die Lockout-Schutzregel, s. u. – NICHT gleichbedeutend mit Sudo-Rechten
    hashedPassword = null; # siehe Erklärung unten
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... <admin-user>"
    ];
  };
}
```

### `modules/baseline/sudo.nix` (Schritt 4)

```nix
# <repo-root>/modules/baseline/sudo.nix
{ ... }:
{
  # security.sudo.wheelNeedsPassword bleibt bewusst beim NixOS-Default
  # `true` – sonst würden alle wheel-Mitglieder passwortlos sudoen,
  # nicht nur <admin-user>. Diese Regel gilt gezielt nur für einen Nutzer.
  security.sudo.extraRules = [
    {
      users = [ "<admin-user>" ];
      commands = [
        {
          command = "ALL";
          options = [ "NOPASSWD" ];
        }
      ];
    }
  ];
}
```

### `modules/baseline/ssh.nix` (Schritt 5)

```nix
# <repo-root>/modules/baseline/ssh.nix
{ ... }:
{
  services.openssh = {
    enable = true;
    ports = [ 40 ]; # <ssh-port>
    openFirewall = true;
    settings = {
      PermitRootLogin = "no";
      # Global an, weil LDAP-Nutzer (Schritt 8) per Passwort einloggen.
      # <admin-user> hat kein Passwort gesetzt und kommt darüber ohnehin nie rein.
      PasswordAuthentication = true;
      KbdInteractiveAuthentication = true;
    };
  };
}
```

### `modules/baseline/fail2ban.nix` (Schritt 7)

```nix
# <repo-root>/modules/baseline/fail2ban.nix
{ ... }:
{
  services.fail2ban = {
    enable = true;
    maxretry = 3;
    bantime = "1h";
    bantime-increment.enable = true;
  };
}
```

### `modules/baseline/ldap.nix` (Schritt 8)

```nix
# <repo-root>/modules/baseline/ldap.nix
{ ... }:
{
  services.sssd = {
    enable = true;
    config = ''
      [sssd]
      services = nss, pam
      domains = ldap

      [nss]

      [pam]

      [domain/ldap]
      id_provider = ldap
      auth_provider = ldap
      ldap_uri = <ldap-uri>
      ldap_search_base = <ldap-base-dn>
      ldap_schema = rfc2307
      cache_credentials = true
      enumerate = false
    '';
  };

  # LDAP-Nutzer stehen nie in users.users – ohne das hier bliebe ihr
  # $HOME beim ersten Login unangelegt.
  security.pam.services.sshd.makeHomeDir = true;
}
```

> ⚠️ Wenn `<ldap-uri>` einen authentifizierten Bind verlangt, gehören `ldap_default_bind_dn`/`ldap_default_authtok` zusätzlich in diese Datei — als Platzhalter, aufgelöst über `services.sssd.environmentFile` und eine Klartextdatei außerhalb des Repos (Schritt 8, Warnbox). Echtes Secrets-Management für diesen Wert gibt es in Projekt 1 nicht; siehe Ausbaustufe 5 unten.

### `modules/baseline/firewall.nix` (Schritt 9)

```nix
# <repo-root>/modules/baseline/firewall.nix
{ ... }:
{
  networking.firewall = {
    enable = true;               # ist ohnehin die Voreinstellung
    allowedTCPPorts = [ <ssh-port> ];
    allowedUDPPorts = [ ];       # nichts – der Runner braucht nur ausgehende Verbindungen
    allowPing = false;
  };
}
```

### `modules/runner/default.nix` (Schritt 10 angelegt, Schritt 11 erweitert)

```nix
# <repo-root>/modules/runner/default.nix
{
  imports = [
    ./container-runtime.nix
    ./forgejo-runner.nix
  ];
}
```

### `modules/runner/container-runtime.nix` (Schritt 10)

```nix
# <repo-root>/modules/runner/container-runtime.nix
{ pkgs, ... }:
{
  virtualisation.podman = {
    enable = true;
    # Erlaubt Job-Containern desselben Workflows (Actions-"services:"-Container),
    # sich per Name statt nur per IP zu erreichen. Öffnet dafür laut
    # nixpkgs nur UDP/53 auf dem eigenen podman0-Interface, nicht global –
    # Schritt 9s "nur Port <ssh-port> eingehend" bleibt unangetastet.
    defaultNetwork.settings.dns_enabled = true;
  };

  # Vorsorglich fuse-overlayfs statt des nativen Overlay-Treibers, siehe
  # Begründung oben (Durchführung, Punkt 2).
  virtualisation.containers.storage.settings.storage.options.mount_program =
    "${pkgs.fuse-overlayfs}/bin/fuse-overlayfs";
}
```

### `modules/runner/forgejo-runner.nix` (Schritt 11 angelegt, Schritt 12 = Endzustand der Hauptvariante)

Schritt 13 registriert nur eine externe Token-Datei — an dieser Datei ändert sich dabei nichts.

```nix
# <repo-root>/modules/runner/forgejo-runner.nix
{ pkgs, utils, ... }:
{
  services.gitea-actions-runner = {
    package = pkgs.forgejo-runner;

    instances."<runner-name>" = {
      enable = true;
      name = "<runner-name>";
      url = "<forgejo-url>";

      # Liegt außerhalb des Repos und außerhalb des Stores; Schritt 13
      # legt sie mit `TOKEN=<registrierungstoken>` an (root:root, 0600).
      # Zwischen Schritt 11 und 13 fehlt sie bewusst – der Dienst baut
      # dann zwar, startet aber nicht.
      tokenFile = "/etc/gitea-runner-<runner-name>-token.env";

      labels = [
        "ubuntu-latest:docker://node:20-bookworm"
      ];
    };
  };

  # utils.escapeSystemdPath berechnet denselben escapten Namen, den das
  # Runner-Modul intern für die Unit verwendet (Schritt 11) – von Hand
  # zusammengebaut wäre er bei jedem Bindestrich in <runner-name> falsch.
  # Als Liste geschrieben, führt der Merge für `unitOption` diesen Eintrag
  # mit dem bereits vorhandenen EnvironmentFile aus instance.tokenFile
  # zusammen, statt einen Konflikt zu melden.
  systemd.services."gitea-runner-${utils.escapeSystemdPath "<runner-name>"}".serviceConfig.EnvironmentFile = [
    "${./runner.env}"
  ];
}
```

### `modules/runner/runner.env` (Schritt 12, Klartext-Variante)

```bash
# <repo-root>/modules/runner/runner.env
RUNNER_ENVIRONMENT=produktion
RUNNER_SITE=rz-intern
TZ=Europe/Berlin
```

### Exkurs: sops-age-Variante (Schritt 15) — Alternative, kein Ersatz

Der Exkurs ersetzt `modules/runner/runner.env` nicht, sondern stellt ihm die folgenden Dateien gegenüber. Er zeigt dabei **zwei** Wege, die sich nur im `sops.age`-Block unterscheiden: den Hauptweg mit deinem bereits vorhandenen age-Schlüssel und die Variante mit getrennten Rollen. `forgejo-runner.nix` existiert damit in insgesamt drei möglichen Fassungen — welche aktiv ist, entscheidet einzig, was tatsächlich in `<repo-root>/modules/runner/forgejo-runner.nix` steht. In keinem der beiden Wege wird ein age-Schlüssel erzeugt.

```yaml
# <repo-root>/.sops.yaml — Hauptweg: dein vorhandener Schluessel als einziger Empfaenger
creation_rules:
  - path_regex: secrets/.*\.env$
    key_groups:
      - age:
          - <age-recipient>
```

In der Variante kommt darunter ein zweiter Empfänger dazu: die `ssh-to-age`-Ausgabe aus dem SSH-Host-Key des Containers. Konkrete `age1…`-Werte stehen hier bewusst nirgends — ein echt aussehender Empfänger ließe sich kommentarlos kopieren.

```bash
# <repo-root>/secrets/runner.env (im Repo nur verschlüsselt abgelegt; hier der Klartext, den `sops` anzeigt)
RUNNER_ENVIRONMENT=produktion
RUNNER_SITE=rz-intern
TZ=Europe/Berlin
```

```nix
# <repo-root>/modules/runner/forgejo-runner.nix — ALTERNATIVE (Exkurs Schritt 15), ersetzt die obige Klartext-Fassung
{ config, pkgs, utils, ... }:
{
  # Hauptweg: dein vorhandener Schluessel, per pct push nach
  # /var/lib/sops-nix/key.txt gebracht (root:root, 0600).
  # sshKeyPaths muss explizit leer sein, sonst haengt sops-nix per
  # Default zusaetzlich den ed25519-Host-Key als zweite Identitaet ein.
  sops.age = {
    keyFile = "/var/lib/sops-nix/key.txt";
    generateKey = false;
    sshKeyPaths = [ ];
  };
  # Variante mit getrennten Rollen -- ersetzt genau den Block darueber:
  #   sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
  # Dann liegt kein privater Schluessel auf dem Container, dafuer steht
  # dessen abgeleitete Identitaet als zweiter Empfaenger in .sops.yaml.
  sops.secrets."runner-env" = {
    sopsFile = ../../secrets/runner.env;
    format = "dotenv";
  };

  services.gitea-actions-runner = {
    package = pkgs.forgejo-runner;

    instances."<runner-name>" = {
      enable = true;
      name = "<runner-name>";
      url = "<forgejo-url>";

      tokenFile = "/etc/gitea-runner-<runner-name>-token.env";

      labels = [
        "ubuntu-latest:docker://node:20-bookworm"
      ];
    };
  };

  systemd.services."gitea-runner-${utils.escapeSystemdPath "<runner-name>"}".serviceConfig.EnvironmentFile = [
    config.sops.secrets."runner-env".path
  ];
}
```

```nix
# <repo-root>/flake.nix — ALTERNATIVE (Exkurs Schritt 15), ersetzt die Hauptvariante oben
{
  description = "Infrastruktur für <hostname> (Forgejo-Runner)";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    sops-nix.url = "github:Mic92/sops-nix";
    sops-nix.inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = { self, nixpkgs, sops-nix, ... }: {
    nixosConfigurations.<hostname> = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./hosts/<hostname>/configuration.nix
        sops-nix.nixosModules.sops
      ];
    };
  };
}
```

> 💡 **Nice to know:** Der Unterschied zwischen den beiden `forgejo-runner.nix`-Fassungen ist kein Detail: `"${./runner.env}"` ist ein Nix-**Pfad**-Literal (Store-Kopie beim Bauen, weltlesbar), `config.sops.secrets."runner-env".path` zur Auswertungszeit nur ein **String**, der erst beim Aktivieren auf `/run/secrets/runner-env` (`tmpfs`) zeigt. Beide Fassungen bauen ohne Fehler — der Unterschied zeigt sich erst darin, wo der Klartext am Ende tatsächlich liegt.

## 3. Was du jetzt kannst

- Einen Proxmox-LXC-Container vollständig deklarativ anlegen: minimales Bootstrap-Template + versioniertes `pct create`-Skript statt Ad-hoc-Klicks in der GUI.
- Das Zusammenspiel von `--ostype unmanaged` und `proxmoxLXC.manageNetwork` durchschauen und für einen zweistufigen Bootstrap (minimal per Template, vollständig per `nixos-rebuild` von innen) gezielt nutzen.
- NixOS-Module in wachsenden Sammel-Imports (`default.nix`) organisieren, bei denen jeder Arbeitsschritt genau eine neue Zeile hinzufügt — eine diff-freundliche, nachvollziehbare Konfigurationsgeschichte.
- Merge-Verhalten listenwertiger Optionen gezielt einsetzen statt nur zu vermeiden: Reihenfolge bei `security.sudo.extraRules` (`mkOrder`), automatischer Listen-Merge bei `systemd.services.*.serviceConfig.EnvironmentFile` (`unitOption`).
- Einen lokalen Fallback-Account sauber von LDAP-Nutzern trennen — `hashedPassword = null` plus `mutableUsers = false` als struktureller Sperrmechanismus für genau ein Konto, ohne eine globale SSH-Einstellung anzufassen.
- `sssd`/LDAP für Login parallel zu deklarativen Nix-Nutzern betreiben und einordnen, warum `users.mutableUsers` daran nichts ändert (separate NSS-Quelle `sss`).
- systemd-Unit-Namen korrekt über `escapeSystemdPath` referenzieren, statt sie aus einem Instanznamen von Hand zusammenzubauen.
- Zugangsdaten korrekt außerhalb des weltlesbaren Nix Store halten (`tokenFile`-Muster mit externer, `chmod 600`-geschützter Datei) und den qualitativen Unterschied zu einer sops-age-Lösung mit `/run/secrets` benennen.
- Eine Container-Runtime (Podman) in einem unprivilegierten, genesteten LXC lauffähig machen — die richtigen Proxmox-Features (`nesting`, `keyctl`, `fuse`) und `fuse-overlayfs` als vorsorgliche statt reaktive Maßnahme.
- Einen Forgejo-Actions-Runner registrieren und die Label-Semantik (`:docker://…` vs. `:host`) sowie die verfügbaren Registrierungs-Scopes (Instanz/Organisation/Nutzer/Repository) einordnen.
- Ein Firewall-Modell konsequent für einen Dienst anwenden, der sich nur aktiv nach außen verbindet (Long-Polling) — eingehend zählt nur, was tatsächlich gebraucht wird.

## 4. Ausbaustufen

1. **Mehrere Runner-Instanzen parallel.** `services.gitea-actions-runner.instances` ist ein `attrsOf submodule` (Schritt 11) — ein zweiter Eintrag mit eigenem Namen, eigenen Labels (z. B. eine andere Node-Version oder ein anderes Basis-Image) und eigener `tokenFile` läuft im selben Container, ohne die Modul-Struktur zu ändern. Sinnvoll, sobald ein Label-Typ zum CI-Engpass wird.
2. **`:host`-Labels statt reiner Container-Jobs.** Ein zusätzliches Label wie `bare:host` in `forgejo-runner.nix` lässt Jobs direkt im Runner-Container laufen statt in einem gepullten Image — kein Pull, dafür explizite Pflege von `instances.<name>.hostPackages` (Default siehe Schritt 11/14). Sinnvoll für Jobs, die nur ohnehin vorhandene Werkzeuge brauchen.
3. **Cache-/Registry-Spiegel im internen Netz.** Schritt 14 musste `node:20-bookworm` manuell mit `podman pull` vorab cachen, weil das Netz laut Projektrahmen ohne öffentlichen Zugriff auskommt. Ein interner OCI-Registry-Spiegel macht diesen manuellen Schritt überflüssig und beschleunigt jeden Neustart mit leerem Storage.
4. **Monitoring des Runners.** Bislang liefert nur `journalctl -u gitea-runner-*` (Schritt 11/13) Einblick. Ein Metriken-Exporter, ergänzt um eine eng gefasste zusätzliche Firewall-Freigabe für den Scrape-Port (Schritt 9 als Vorlage: gezielt ein weiterer `allowedTCPPorts`-Eintrag statt einer pauschalen Öffnung), macht Jobdauer und Neustart-Zyklen sichtbar, statt sie erst im Fehlerfall zu suchen.
5. **Deklaratives Secrets-Management für alle Werte.** Schritt 15 verschlüsselt nur `runner.env`; das LDAP-Bind-Passwort (Schritt 8, Warnbox zu `environmentFile`) und die Token-Datei (Schritt 13) liegen weiterhin als Klartext außerhalb des Store. Beide ließen sich nach demselben sops-age-Muster (eigener `sops.secrets`-Eintrag, Ziel-Option auf `config.sops.secrets."…".path`) verschlüsseln — der konsequente nächste Schritt, den Projekt 2 laut `ENTSCHEIDUNGEN.md` für echte Zugangsdaten ohnehin geht. Wer diese Stufe zündet, sollte zugleich vom Hauptweg auf die Variante mit getrennten Rollen wechseln: Je mehr Secrets an einem einzigen, auf dem Host liegenden Schlüssel hängen, desto teurer wird ein kompromittierter Host — und spätestens beim zweiten Host teilen sich beide denselben Generalschlüssel, was die Trennung zwischen ihnen aufhebt.

## 5. Teardown

Vollständiger Rückbau, in der Reihenfolge, die am wenigsten verwaiste Zustände hinterlässt: zuerst den Dienst beruhigen, dann die Forgejo-Seite bereinigen, dann lokale Spuren im Container entfernen, zuletzt Container und Template auf dem Proxmox-Host beseitigen.

**1. Runner-Dienst stoppen** (verhindert, dass der Runner während der folgenden Schritte noch Jobs annimmt oder sich neu registriert):

```console
$ pct exec <vmid> -- systemctl stop "gitea-runner-$(systemd-escape '<runner-name>')"
```

**2. Runner-Eintrag in Forgejo löschen.** In derselben Runner-Übersicht, in der Schritt 13 den Runner angelegt hat (instanzweit `/admin/actions/runners` bei entsprechendem Scope) — ohne diesen Schritt bleibt der Runner dort dauerhaft als "offline" gelistet, auch nachdem Container und Token längst verschwunden sind.

> ⚠️ Ungeprüft: Der exakte UI-Pfad zum Löschen eines Runner-Eintrags wurde ebenso wenig direkt an der Forgejo-Oberfläche verifiziert wie der Registrierungspfad in Schritt 13 (`forgejo.org` war aus der Recherche-Umgebung nicht erreichbar) — plausibel dieselbe Übersichtsseite, die auch die Registrierung zeigt, aber vor Gebrauch gegenprüfen.

**3. Token-Datei und Laufzeitzustand im Container löschen:**

```console
$ pct exec <vmid> -- rm -f /etc/gitea-runner-<runner-name>-token.env
$ pct exec <vmid> -- rm -rf /var/lib/gitea-runner/<runner-name>
```

**4. Container stoppen und zerstören** (Proxmox-Host):

```console
$ pct stop <vmid>
$ pct destroy <vmid>
```

**5. Template-Datei aus dem Storage-Cache entfernen** (Proxmox-Host):

```console
$ rm /var/lib/vz/template/cache/nixos-<hostname>-bootstrap.tar.xz
```

**6. LXC-Features zurücksetzen.** Aus Schritt 10s Rückweg dokumentiert, hier der Vollständigkeit halber: `pct set <vmid> --features nesting=0,keyctl=0,fuse=0` gefolgt von `pct reboot <vmid>`. Praktisch ist dieser Befehl nach Schritt 4 bereits gegenstandslos — `pct destroy` entfernt die komplette Container-Konfiguration inklusive aller Feature-Flags. Relevant wird er nur in einem abweichenden Szenario: Wenn `<vmid>` **nicht** zerstört, sondern nur auf den Stand vor Schritt 10 zurückgesetzt werden soll (z. B. um denselben Container ohne Runner weiterzuverwenden), gehört dieser Befehl **vor** einen eventuellen `pct destroy` — dann muss der Container beim Ausführen noch existieren.

Von `<repo-root>` selbst nimmt dieser Teardown nichts weg: Das Repo enthält an keiner Stelle Zugangsdaten (Token, LDAP-Bind-Passwort liegen beide außerhalb, siehe Abschnitt 1) und lässt sich unverändert für einen neuen Container-Anlauf wiederverwenden. `secrets/runner.env` und `.sops.yaml` aus dem Exkurs dürfen ebenfalls liegen bleiben — verschlüsselt und ohne Bezug zu diesem Container. Wurde der Hauptweg nachgebaut, verschwindet mit `pct destroy` auch die Kopie deines privaten age-Schlüssels unter `/var/lib/sops-nix/key.txt`; das Original auf deiner Workstation (`<age-key-file>`) bleibt davon selbstverständlich unberührt und wird für den nächsten Anlauf wieder gebraucht.
