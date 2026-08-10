---
title: "Exkurs: sops-age"
weight: 15
---

# Schritt 15: Exkurs — dieselbe `.env` sops-age-verschlüsselt, mit dem Schlüssel, den du schon hast

## Ziel

`<repo-root>/secrets/runner.env` liegt sops-age-verschlüsselt neben dem Klartext aus Schritt 12: im Hauptweg entschlüsselt durch deinen age-Schlüssel, in einer Variante durch einen aus dem SSH-Host-Key abgeleiteten Zweitschlüssel; `forgejo-runner.nix` zeigt am Ende auf `/run/secrets/…`. Dasselbe Muster schließt danach die Lücke aus Schritt 8: `secrets/sssd.env` verschlüsselt LDAP-URI, Bind-DN und Bind-Passwort, `ldap.nix` zeigt über `services.sssd.environmentFile` darauf.

## Voraussetzung

Teil I, Kapitel 10 ist gelesen – sops-nix, age vs. GPG, `/run/secrets/…` werden vorausgesetzt. Du besitzt ein age-Schlüsselpaar (`<age-recipient>`, `<age-key-file>`) – dieser Exkurs erzeugt keins, weder per `age-keygen` noch über `sops.age.generateKey`. Schritt 5 (SSH-Host-Key, nur Variante) und 12/13 sind abgeschlossen: `forgejo-runner.nix` hat seit Schritt 12 `{ pkgs, utils, ... }:` im Kopf und `EnvironmentFile = [ "${./runner.env}" ];`, gemergt mit `tokenFile` aus Schritt 11. Schritt 8 ist abgeschlossen: `ldap.nix` bindet mit `ldap_default_bind_dn`/`ldap_default_authtok` im Klartext – die Lücke aus dessen Warnbox.

## Durchführung

**1. sops-nix als Flake-Input.** `github:Mic92/sops-nix` hat keine versionierten Release-Tags – wie bei `nixpkgs` in Schritt 2 pinnt nicht der Branchname, sondern `flake.lock`.

**2. `.sops.yaml` mit `<age-recipient>` als Empfänger anlegen**, `secrets/runner.env` verschlüsseln (dotenv-Format explizit):

```console
$ sops --input-type dotenv --output-type dotenv secrets/runner.env
```

**3. Den privaten Schlüssel auf den Container bringen.** Der heikle Teil: `<age-key-file>` vom Proxmox-Host auf den Container, Muster wie Schritt 2/13 (`pct push`, Zielverzeichnis, `chown`, `chmod`):

```console
$ pct exec <vmid> -- mkdir -p /var/lib/sops-nix
$ pct push <vmid> <age-key-file> /var/lib/sops-nix/key.txt
$ pct exec <vmid> -- chown root:root /var/lib/sops-nix/key.txt
$ pct exec <vmid> -- chmod 600 /var/lib/sops-nix/key.txt
```

Der Schlüssel gehört nicht ins Repo – anders als der öffentliche Empfänger – und nicht als Pfad-Literal (`./key.txt`, Store-Kopie beim Bauen). Kein Stilhinweis: `keyFile` ist als `lib.types.nullOr pathNotInStore` typisiert;<sup>3</sup> einen Store-Pfad lehnt das Modulsystem zur Auswertungszeit ab.

**4. `forgejo-runner.nix` erweitern**, pushen, rebuilden (Diff siehe "Dateien").

**Variante — getrennte Rollen (kompakt).** Der Schlüssel bleibt auf der Workstation; der Container entschlüsselt über eine aus seinem SSH-Host-Key abgeleitete Identität:

```console
$ pct exec <vmid> -- cat /etc/ssh/ssh_host_ed25519_key.pub | \
    nix shell nixpkgs#ssh-to-age -c ssh-to-age
```

Die Ausgabe wird zweiter Empfänger neben `<age-recipient>` (Diff siehe "Dateien"), danach bereits verschlüsselte Dateien nachziehen:<sup>4</sup>

```console
$ sops updatekeys secrets/runner.env
```

`forgejo-runner.nix` setzt in dieser Variante `sops.age.sshKeyPaths` statt `keyFile`/`generateKey` (Diff siehe "Dateien").

**Die Abwägung dahinter.** Der Hauptweg hat einen Empfänger, kein `sops updatekeys` beim Neuaufsetzen, ist am einfachsten – der Preis: Der private Schlüssel liegt auf jedem Zielsystem, das ihn nutzt, und wer dort Root hat, hat damit Zugriff auf *alle* damit verschlüsselten Secrets, nicht nur die dieses Containers. Die Variante kostet mehr Pflege – zwei Empfänger, `sops updatekeys` nach jedem Neuaufsetzen –, aber der Schlüssel verlässt die Workstation nie; ein kompromittierter Container gibt nur seine eigene Identität preis. Für einen Host wie hier ist der Hauptweg vertretbar; bei mehreren Hosts mit demselben Empfänger kippt das – Projekt 2 und 3 gehen den Weg der Variante.

**5. Zweiter Fall: das Bind-Passwort aus Schritt 8.** `<ldap-uri>` und `<ldap-base-dn>` sind keine Geheimnisse im kryptografischen Sinn – in jeder LDAP-Umgebung ohnehin auffindbar, im Buch ohnehin nur Platzhalter. Das Bind-Passwort dagegen ist eines: das Secret, um das es der Warnbox in Schritt 8 eigentlich ging. Alle drei trotzdem gemeinsam aus dem Store zu halten hat einen dritten Grund: Sie sind standortspezifisch, das Repo soll die Topologie der eigenen Umgebung nicht nach außen tragen. Das Muster von Punkt 1–4 bleibt unverändert – verschlüsseltes `secrets/*.env`, ein `sops.secrets`-Eintrag, `sops.age` bereits systemweit gesetzt (Punkt 4), `flake.nix` unverändert. Anders ist nur das Ziel: statt `EnvironmentFile` am Runner-Unit `services.sssd.environmentFile`, mit Platzhaltern mitten in der `sssd.conf` (Diff siehe "Dateien").

## Dateien

`<repo-root>/.sops.yaml` (neu, Hauptweg):

```yaml
# <repo-root>/.sops.yaml
creation_rules:
  - path_regex: secrets/.*\.env$
    key_groups:
      - age:
          - <age-recipient>
```

`<repo-root>/secrets/runner.env` (neu, verschlüsselt – Klartext identisch mit Schritt 12):

```bash
RUNNER_ENVIRONMENT=produktion
RUNNER_SITE=rz-intern
TZ=Europe/Berlin
```

Im Repo liegt nur die verschlüsselte Fassung. `dotenv` verschlüsselt nur die *Werte* – Variablennamen bleiben lesbar, ein `git diff` zeigt also, *welcher* Eintrag sich änderte, nicht wie. `format = "binary"` wäre ein undurchsichtiger Block; möglich, kostet aber diese Lesbarkeit.

> 💡 **Nice to know:** Die Options-Beschreibung von `sops.secrets.<name>.key` legt eine Falle aus: Sie sagt, der Schlüssel werde "in der sops-Datei nachgeschlagen", der Default sei der Name des Secrets, und `""` bedeute "whole file" – man könnte also meinen, hier müsse zwingend `key = "";` stehen, sonst suche sops-nix einen Eintrag namens `runner-env` *innerhalb* der Datei. Für `dotenv` stimmt das nicht: Im Go-Quellcode landet `dotenv` – zusammen mit `binary` und `ini` – in dem Zweig, der immer den gesamten entschlüsselten Inhalt übernimmt, und die Schlüsselprüfung überspringt diese drei Formate ausdrücklich. `key` ist hier also wirkungslos, nicht Pflicht. Nachgeschlagen wird nur bei `yaml` und `json`.<sup>2</sup>

`<repo-root>/modules/runner/forgejo-runner.nix` (geändert, Hauptweg):

```diff
--- a/modules/runner/forgejo-runner.nix
+++ b/modules/runner/forgejo-runner.nix
@@
-{ pkgs, utils, ... }:
+{ config, pkgs, utils, ... }:
 {
+  sops.age = {
+    keyFile = "/var/lib/sops-nix/key.txt";
+    generateKey = false;
+    sshKeyPaths = [ ];
+  };
+  sops.secrets."runner-env" = {
+    sopsFile = ../../secrets/runner.env;
+    format = "dotenv";
+  };
+
   services.gitea-actions-runner = {
     package = pkgs.forgejo-runner;
     instances."<runner-name>" = { … };
   };

   systemd.services."gitea-runner-${utils.escapeSystemdPath "<runner-name>"}".serviceConfig.EnvironmentFile = [
-    "${./runner.env}"
+    config.sops.secrets."runner-env".path
   ];
 }
```

Die drei Zeilen unter `sops.age`, einzeln:

- **`keyFile`** – Pfad aus Punkt 3, String wie `tokenFile` in Schritt 13, kein Pfad-Literal.
- **`generateKey = false;`** – bereits der Default ("key must already be present at the specified location"<sup>3</sup>), hier nur zur Klarheit explizit.
- **`sshKeyPaths = [ ];`** – muss explizit leer sein: Default sind die ed25519-Keys aus `config.services.openssh.hostKeys`;<sup>3</sup> sonst hängt sops-nix den Host-Key zusätzlich zu `keyFile` ein, und "nur mein Schlüssel zählt" wäre falsch.

`sops.secrets."runner-env"` braucht kein eigenes `mode`/`owner` – Default (`root:root`, `0400`) genügt, wie `tokenFile` in Schritt 13: `EnvironmentFile=` liest root, bevor er zu `DynamicUser`-Nutzer `gitea-runner` wechselt.

`<repo-root>/flake.nix` (geändert):

```diff
--- a/flake.nix
+++ b/flake.nix
@@
   inputs = {
     nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
+    sops-nix.url = "github:Mic92/sops-nix";
+    sops-nix.inputs.nixpkgs.follows = "nixpkgs";
   };
-  outputs = { self, nixpkgs, ... }: {
+  outputs = { self, nixpkgs, sops-nix, ... }: {
     nixosConfigurations.<hostname> = nixpkgs.lib.nixosSystem {
       system = "x86_64-linux";
-      modules = [ ./hosts/<hostname>/configuration.nix ];
+      modules = [
+        ./hosts/<hostname>/configuration.nix
+        sops-nix.nixosModules.sops
+      ];
     };
   };
 }
```

Dasselbe Prinzip wie bei `keyFile` oben, für die `.env`: `./runner.env` kopiert als Pfad-Literal in den Store; `config.sops.secrets."runner-env".path` bleibt bis zur Aktivierung ein String, danach zeigt er auf `/run/secrets/runner-env`.<sup>1</sup> `flake.nix` bleibt in der Variante unverändert.

### Variante — getrennte Rollen: die abweichenden Stellen

`<repo-root>/.sops.yaml` (geändert, gegenüber dem Hauptweg oben):

```diff
--- a/.sops.yaml
+++ b/.sops.yaml
@@
     key_groups:
       - age:
           - <age-recipient>
+          # Zusaetzlich: Ausgabe von `ssh-to-age` (Durchfuehrung, Variante)
+          # aus dem SSH-Host-Key des jeweiligen Containers.
```

`<repo-root>/modules/runner/forgejo-runner.nix` (geändert gegenüber der Hauptweg-Fassung – nur der `sops.age`-Block wechselt):

```diff
--- a/modules/runner/forgejo-runner.nix
+++ b/modules/runner/forgejo-runner.nix
@@
-  sops.age = {
-    keyFile = "/var/lib/sops-nix/key.txt";
-    generateKey = false;
-    sshKeyPaths = [ ];
-  };
+  sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
   sops.secrets."runner-env" = {
     sopsFile = ../../secrets/runner.env;
     format = "dotenv";
   };
```

Kein `keyFile`, kein privater Schlüssel im Container – `sshKeyPaths` genügt, der Host-Key liegt schon dort (Schritt 5).

### Zweiter Fall: LDAP-Bind-Passwort

`<repo-root>/.sops.yaml` bleibt unverändert – `path_regex: secrets/.*\.env$` erfasst auch die neue Datei:

```console
$ sops --input-type dotenv --output-type dotenv secrets/sssd.env
```

`<repo-root>/secrets/sssd.env` (neu, verschlüsselt):

```bash
SSSD_LDAP_URI=<ldap-uri>
SSSD_LDAP_SEARCH_BASE=<ldap-base-dn>
SSSD_LDAP_DEFAULT_BIND_DN=cn=svc-bind,<ldap-base-dn>
SSSD_LDAP_DEFAULT_AUTHTOK=<bind-passwort>
```

`SSSD_LDAP_DEFAULT_BIND_DN` steht hier nur beispielhaft für einen Service-Account unterhalb von `<ldap-base-dn>` – der tatsächliche DN hängt vom jeweiligen LDAP-Server ab.

`<repo-root>/modules/baseline/ldap.nix` (geändert):

```diff
--- a/modules/baseline/ldap.nix
+++ b/modules/baseline/ldap.nix
@@
-{ ... }:
+{ config, ... }:
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
-      ldap_uri = <ldap-uri>
-      ldap_search_base = <ldap-base-dn>
+      ldap_uri = $SSSD_LDAP_URI
+      ldap_search_base = $SSSD_LDAP_SEARCH_BASE
+      ldap_default_bind_dn = $SSSD_LDAP_DEFAULT_BIND_DN
+      ldap_default_authtok = $SSSD_LDAP_DEFAULT_AUTHTOK
       ldap_schema = rfc2307
       cache_credentials = true
       enumerate = false
     '';
   };
+  services.sssd.environmentFile = config.sops.secrets."sssd-env".path;
+
+  sops.secrets."sssd-env" = {
+    sopsFile = ../../secrets/sssd.env;
+    format = "dotenv";
+    restartUnits = [ "sssd.service" ];
+  };

   # LDAP-Nutzer stehen nie in users.users – ohne das hier bliebe ihr
   # $HOME beim ersten Login unangelegt.
   security.pam.services.sshd.makeHomeDir = true;
 }
```

`services.sssd.environmentFile` landet im Unit als `EnvironmentFile=`, nicht als systemd-Credential – der Quellcode kommentiert das selbst: "We cannot use LoadCredential here because it's not available in ExecStartPre"<sup>5</sup>: `preStart` ersetzt die Platzhalter per `envsubst`, bevor der eigentliche Programmstart beginnt, `LoadCredential` steht dort noch nicht zur Verfügung.

`restartUnits = [ "sssd.service" ];` sorgt dafür, dass ein geändertes Passwort den laufenden Dienst erreicht – ohne das bliebe die alte Version bis zum nächsten manuellen Neustart aktiv, das Verhalten folgt `systemd.services.<name>.restartTriggers`.<sup>6</sup> Eine eigene `after`-Verdrahtung zu `sssd.service` braucht es nicht: `sops-install-secrets` läuft schon vor regulären Diensten (`wantedBy = [ "sysinit.target" ]`, zusätzlich ein `activationScripts`-Eintrag beim `switch`).<sup>6</sup> Owner/Mode bleiben Default (`root:root`, `0400`) – `sssd` läuft als root, kein `DynamicUser`-Sonderfall wie beim Runner.

## Prüfen

- `pct exec <vmid> -- cat /run/secrets/runner-env` zeigt denselben Klartext wie beim `sops`-Aufruf; `findmnt /run/secrets` weist das Ziel als `tmpfs` aus. Gilt für beide Varianten.
- **Hauptweg:** `pct exec <vmid> -- stat -c '%a %U:%G' /var/lib/sops-nix/key.txt` zeigt `600 root:root` – fehlt die Datei, wurde Punkt 3 übersprungen oder die Variante ist aktiv.
- `pct exec <vmid> -- grep -rl "RUNNER_SITE" /run/current-system` liefert **keinen** Treffer mehr: Nur der Pfad wird referenziert; der alte Store-Pfad bleibt bis `nix-collect-garbage` liegen – die neue Fassung landet dort **nie**.
- `pct exec <vmid> -- systemctl cat 'gitea-runner-*'` zeigt die `EnvironmentFile=`-Zeile aus Schritt 13 unverändert sowie eine zweite, jetzt auf `/run/secrets/runner-env`.
- Der Runner-Dienst bleibt aktiv; `journalctl` zeigt keinen neuen Fehler.
- **Zweiter Fall:** Store-Fassung enthält nur den Platzhalter: `pct exec <vmid> -- grep -rl SSSD_LDAP_DEFAULT_AUTHTOK /nix/store` findet die unsubstituierte `sssd.conf`. Laufzeit-Fassung: `pct exec <vmid> -- grep ldap_default_authtok /var/lib/sssd/sssd.conf` zeigt den echten Wert, `stat -c '%a' /var/lib/sssd/sssd.conf` liefert `600` – geschrieben von `preStart` unter `umask 0177`, außerhalb des Stores.<sup>5</sup>
- Login aus Schritt 8 (`ssh -p <ssh-port> testuser@<ip>`) funktioniert unverändert weiter.

## Wenn's schiefgeht

**`Failed to get the data key required to decrypt the SOPS file.`** – wie in Kapitel 10: Im Hauptweg zeigt `keyFile` auf die falsche/fehlende Datei, oder `<age-recipient>` fehlt in `.sops.yaml`. In der Variante zeigt `sshKeyPaths` auf den falschen Host-Key, oder dessen `ssh-to-age`-Ausgabe fehlt als Empfänger. Gegenprüfen, dann `sops updatekeys secrets/runner.env`.

**Build bricht mit *"No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home"* ab:** Nur im Hauptweg möglich – `sshKeyPaths = [ ]` übernommen, `keyFile` vergessen. Ohne Empfänger verweigert das Modul den Build per Assertion.<sup>3</sup> Fix: `keyFile` ergänzen.

**Build bricht mit `error: undefined variable 'config'` ab:** Funktionskopf nicht um `config` erweitert – nötig in beiden Varianten wegen `config.sops.secrets."runner-env".path`. Fix: Kopfzeile wie im Diff korrigieren.

**Build bricht mit `error: undefined variable 'SSSD_LDAP_DEFAULT_AUTHTOK'` ab:** In `ldap.nix` `${SSSD_LDAP_DEFAULT_AUTHTOK}` statt `$SSSD_LDAP_DEFAULT_AUTHTOK` geschrieben – aus Shell-Gewohnheit naheliegend, aber `${...}` ist in einem `''…''`-Block Nix-Interpolation, keine Shell-Syntax. Nix versucht `SSSD_LDAP_DEFAULT_AUTHTOK` als eigene Variable auszuwerten und scheitert, lange bevor `envsubst` überhaupt läuft. Fix: Klammern weg, bare `$VAR` wie im Diff – exakt das Muster aus der Options-Beschreibung von `environmentFile`.<sup>5</sup>

**Login scheitert ohne aussagekräftige Meldung, `journalctl -u sssd` zeigt nichts Auffälliges:** `secrets/sssd.env` fehlt eine Variable oder ihr Wert ist leer. `envsubst` ersetzt sie dann stillschweigend durch nichts, sssd bindet mit leerem Passwort, der LDAP-Server lehnt ab, ohne dass sssd das als Konfigurationsfehler meldet – die unangenehmste Fehlerart hier. Gegenprüfen mit `sops secrets/sssd.env` (entschlüsselt anzeigen) auf fehlende/leere Zeilen.

## Rückweg

`EnvironmentFile`-Liste zurück auf `[ "${./runner.env}" ]`, `sops.*`-Zeilen und `config`-Argument entfernen, `sops-nix.nixosModules.sops` aus `flake.nix` streichen, rebuilden. Im Hauptweg zusätzlich `pct exec <vmid> -- rm -rf /var/lib/sops-nix`. `secrets/runner.env`/`.sops.yaml` können gefahrlos im Repo bleiben. Zweiter Fall zusätzlich: `services.sssd.environmentFile`- und `sops.secrets."sssd-env"`-Zeilen aus `ldap.nix` entfernen, `ldap_default_bind_dn`/`ldap_default_authtok` auf Klartext zurücksetzen (oder ganz weg, falls anonymer Bind reicht), rebuilden.

## Querverweis

sops-nix: Kapitel 10. SSH-Host-Key: Schritt 5. `pct push`: Schritt 2. Klartext-Gegenstück: Schritt 12, Token-Datei: Schritt 13. LDAP-Bind-Warnbox: Schritt 8. Projekt 2/3 übernehmen die Variante über mehrere Hosts hinweg.

---

<sup>1</sup> Nix-Pfad-Literale vs. Strings: Teil I, Kapitel 3/2. `/run/secrets/…` als `tmpfs`-Ziel: [sops-nix – GitHub](https://github.com/Mic92/sops-nix), bereits Kapitel 10 zitiert.

<sup>2</sup> Quelle: sops-nix, `pkgs/sops-install-secrets/main.go`, `decryptSecret`: `case Binary, Dotenv, Ini:` liefert immer die ganze Datei, nur `Yaml`/`JSON` gehen über `recurseSecretKey`; `validateSopsFile` prüft den Schlüssel nur, wenn `Format != Binary && != Dotenv && != Ini`. https://github.com/Mic92/sops-nix/blob/master/pkgs/sops-install-secrets/main.go

<sup>3</sup> Quelle: sops-nix, `modules/sops/default.nix`. `age.keyFile`: `type = lib.types.nullOr pathNotInStore;`. `age.generateKey`: `default = false;`, "the key must already be present at the specified location." `age.sshKeyPaths`-Default: ed25519-Keys aus `config.services.openssh.hostKeys`. Assertion: "No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home". https://github.com/Mic92/sops-nix/blob/master/modules/sops/default.nix

<sup>4</sup> Quelle: `sops`-Quellcode (nicht sops-nix), `cmd/sops/main.go`, Befehl `updatekeys`: "update the keys of SOPS files using the config file". https://github.com/getsops/sops/blob/main/cmd/sops/main.go

<sup>5</sup> Quelle: nixpkgs, `nixos/modules/services/misc/sssd.nix`. `environmentFile`: `type = lib.types.nullOr lib.types.path;`, `default = null;`, Options-Beschreibung mit dem Muster wörtlich: "Secrets may be passed to the service without adding them to the world-readable Nix store, by specifying placeholder variables as the option value in Nix and setting these variables accordingly in the environment file", Beispiel `ldap_default_authtok = $SSSD_LDAP_DEFAULT_AUTHTOK` / `SSSD_LDAP_DEFAULT_AUTHTOK=verysecretpassword`. Unit: `EnvironmentFile = lib.mkIf (cfg.environmentFile != null) cfg.environmentFile;`, Kommentar "We cannot use LoadCredential here because it's not available in ExecStartPre". `preStart`: `mkdir -p "${dataDir}/conf.d"`, dann unter `umask 0177` `${pkgs.envsubst}/bin/envsubst -o ${settingsFile} -i ${settingsFileUnsubstituted}`; `dataDir = "/var/lib/sssd"`, `settingsFile = "${dataDir}/sssd.conf"`. https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/misc/sssd.nix

<sup>6</sup> Quelle: sops-nix, `modules/sops/default.nix`. `sops.secrets.<name>.restartUnits`: "works the same way as systemd.services.<name>.restartTriggers". `systemd.services.sops-install-secrets`: `wantedBy = [ "sysinit.target" ]`, zusätzlich ein `system.activationScripts`-Eintrag beim `switch`. https://github.com/Mic92/sops-nix/blob/master/modules/sops/default.nix
