---
title: "Exkurs: sops-age"
weight: 15
---

# Schritt 15: Exkurs — dieselbe `.env` sops-age-verschlüsselt, mit dem Schlüssel, den du schon hast

## Ziel

`<repo-root>/secrets/runner.env` liegt sops-age-verschlüsselt neben dem Klartext `modules/runner/runner.env` aus Schritt 12 – kein Duplikat, sondern eigener, tatsächlich geheimer Inhalt; `forgejo-runner.nix` bindet am Ende beide gleichzeitig. Im Hauptweg entschlüsselt dein age-Schlüssel, in einer Variante ein aus dem SSH-Host-Key abgeleiteter Zweitschlüssel. Dasselbe Muster schließt danach zwei weitere Lücken: das LDAP-Bind-Passwort aus Schritt 8 und das Registrierungs-Token aus Schritt 13.

## Voraussetzung

Teil I, Kapitel 10 ist gelesen – sops-nix, age vs. GPG, `/run/secrets/…` werden vorausgesetzt. Du besitzt ein age-Schlüsselpaar (`<age-recipient>`, `<age-key-file>`) – dieser Exkurs erzeugt keins. Schritt 5 (SSH-Host-Key, nur Variante) und 12/13 sind abgeschlossen: `forgejo-runner.nix` hat seit Schritt 12 `{ pkgs, utils, ... }:` im Kopf, `EnvironmentFile = [ "${./runner.env}" ];`, gemergt mit `tokenFile` aus Schritt 11. Schritt 8 ist abgeschlossen: `ldap.nix` bindet mit `ldap_default_bind_dn`/`ldap_default_authtok` im Klartext – die Lücke aus dessen Warnbox.

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

**Die Abwägung dahinter.** Der Hauptweg hat einen Empfänger, kein `sops updatekeys` beim Neuaufsetzen – der Preis: Der private Schlüssel liegt auf jedem Zielsystem, das ihn nutzt, und wer dort Root hat, hat Zugriff auf *alle* damit verschlüsselten Secrets, nicht nur die dieses Containers. Die Variante kostet mehr Pflege, aber der Schlüssel verlässt die Workstation nie; ein kompromittierter Container gibt nur seine eigene Identität preis. Für einen Host wie hier ist der Hauptweg vertretbar; bei mehreren Hosts mit demselben Empfänger kippt das – Projekt 2/3 gehen die Variante.

**5. Zweiter Fall: das Bind-Passwort aus Schritt 8.** `<ldap-uri>` und `<ldap-base-dn>` sind keine Geheimnisse im kryptografischen Sinn – in jeder LDAP-Umgebung auffindbar, im Buch nur Platzhalter. Das Bind-Passwort dagegen ist eines: das Secret aus der Warnbox in Schritt 8. Alle drei bleiben trotzdem gemeinsam aus dem Store – standortspezifisch, soll das Repo die Topologie der eigenen Umgebung nicht preisgeben. Das Muster von Punkt 1–4 bleibt; anders ist nur das Ziel: statt `EnvironmentFile` am Runner-Unit `services.sssd.environmentFile` mit Platzhaltern in der `sssd.conf` (Diff siehe "Dateien").

**6. Dritter Fall: das Token aus Schritt 13.** Schritt 13 bleibt korrekt – der Weg ganz ohne sops. Dieser Exkurs bietet eine Alternative im selben Muster: `tokenFile` zeigt statt auf `/etc/gitea-runner-<runner-name>-token.env` auf `config.sops.secrets."runner-token".path` (Diff siehe "Dateien").

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

`<repo-root>/secrets/runner.env` (neu, verschlüsselt – anderer Inhalt als `modules/runner/runner.env`):

```bash
REGISTRY_MIRROR_USER=ci-mirror
REGISTRY_MIRROR_PASSWORD=<registry-mirror-passwort>
```

Zugangsdaten für den in Schritt 14 erwähnten internen Registry-Spiegel – ein plausibles Beispiel für einen echten Geheimwert. `RUNNER_ENVIRONMENT`/`RUNNER_SITE`/`TZ` aus Schritt 12 bleiben unverändert in der Klartextdatei: unkritisch, im `git diff` lesbar. Kriterium: unkritisch → Klartextdatei, geheim → sops – nicht "alles, was mit der Umgebung zu tun hat". Im Repo liegt von `secrets/runner.env` nur die verschlüsselte Fassung; `dotenv` verschlüsselt nur die *Werte*.

> 💡 **Nice to know:** Die Options-Beschreibung von `sops.secrets.<name>.key` legt eine Falle aus: Der Schlüssel werde "in der sops-Datei nachgeschlagen", Default sei der Name des Secrets, `""` bedeute "whole file" – man könnte meinen, hier müsse zwingend `key = "";` stehen, sonst suche sops-nix einen Eintrag namens `runner-env` *innerhalb* der Datei. Für `dotenv` stimmt das nicht: Im Go-Quellcode landet `dotenv` – mit `binary` und `ini` – im Zweig, der immer den gesamten entschlüsselten Inhalt übernimmt; die Schlüsselprüfung überspringt diese drei Formate. `key` ist hier wirkungslos – nachgeschlagen wird nur bei `yaml`/`json`.<sup>2</sup>

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
+  sops.secrets."runner-token" = {
+    sopsFile = ../../secrets/runner-token.env;
+    format = "dotenv";
+    restartUnits = [ "gitea-runner-${utils.escapeSystemdPath "<runner-name>"}.service" ];
+  };
+
   services.gitea-actions-runner = {
     package = pkgs.forgejo-runner;
     instances."<runner-name>" = {
       enable = true;
       name = "<runner-name>";
       url = "<forgejo-url>";
-      tokenFile = "/etc/gitea-runner-<runner-name>-token.env";
+      tokenFile = config.sops.secrets."runner-token".path;
       labels = [ "ubuntu-latest:docker://node:20-bookworm" ];
     };
   };

   systemd.services."gitea-runner-${utils.escapeSystemdPath "<runner-name>"}".serviceConfig.EnvironmentFile = [
     "${./runner.env}"
+    config.sops.secrets."runner-env".path
   ];
 }
```

Die drei Zeilen unter `sops.age`, einzeln:

- **`keyFile`** – Pfad aus Punkt 3, String wie `tokenFile` in Schritt 13, kein Pfad-Literal.
- **`generateKey = false;`** – bereits der Default ("key must already be present at the specified location"<sup>3</sup>), hier nur zur Klarheit explizit.
- **`sshKeyPaths = [ ];`** – muss explizit leer sein: Default sind die ed25519-Keys aus `config.services.openssh.hostKeys`;<sup>3</sup> sonst hängt sops-nix den Host-Key zusätzlich zu `keyFile` ein, und "nur mein Schlüssel zählt" wäre falsch.

Beide `sops.secrets`-Einträge brauchen kein eigenes `mode`/`owner` – Default (`root:root`, `0400`) genügt, wie `tokenFile` in Schritt 13. `"runner-token"` bekommt zusätzlich `restartUnits`: ohne das erreicht ein in Forgejo erneuertes Token den laufenden Dienst nicht.

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

Dasselbe Prinzip wie bei `keyFile` oben, für die `.env`: `./runner.env` kopiert als Pfad-Literal in den Store; `config.sops.secrets."runner-env".path` bleibt bis zur Aktivierung ein String, danach zeigt er auf `/run/secrets/runner-env`.<sup>1</sup> Beide Zeilen bleiben nebeneinander in der `EnvironmentFile`-Liste – unterschiedliche Variablennamen, kein Konflikt. Bei gleichem Namen gilt: mehrere Dateien werden der Reihe nach gelesen, "the later setting will override the earlier setting"<sup>7</sup> – nützlich zum gezielten Überschreiben, eine Falle bei Tippfehlern. `flake.nix` bleibt in der Variante unverändert.

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
+    environmentFile = config.sops.secrets."sssd-env".path;
   };
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

`services.sssd.environmentFile` landet im Unit als `EnvironmentFile=`, nicht als systemd-Credential – der Quellcode kommentiert das selbst: "We cannot use LoadCredential here because it's not available in ExecStartPre"<sup>5</sup>: `preStart` ersetzt die Platzhalter per `envsubst`, bevor `LoadCredential` überhaupt zur Verfügung stünde.

`restartUnits = [ "sssd.service" ];` sorgt dafür, dass ein geändertes Passwort den laufenden Dienst erreicht, analog zu `restartTriggers`.<sup>6</sup> Eine eigene `after`-Verdrahtung braucht es nicht: `sops-install-secrets` läuft schon vor regulären Diensten (`sysinit.target`, plus `activationScripts` beim `switch`).<sup>6</sup> Owner/Mode bleiben Default – `sssd` läuft als root, kein `DynamicUser`-Fall wie beim Runner.

### Dritter Fall: Registrierungs-Token aus Schritt 13

`<repo-root>/secrets/runner-token.env` (neu, verschlüsselt – ersetzt die manuell angelegte `/etc/gitea-runner-<runner-name>-token.env`):

```bash
TOKEN=<token-aus-weboberflaeche>
```

`.sops.yaml` bleibt unverändert, `sops.secrets."runner-token"` und die geänderte `tokenFile`-Zeile stehen bereits im Hauptweg-Diff oben. Schritt 13 wird dadurch nicht falsch – er zeigt den Weg ohne sops, dieser Exkurs ersetzt ihn optional.

## Prüfen

- `pct exec <vmid> -- cat /run/secrets/runner-env` zeigt denselben Klartext wie beim `sops`-Aufruf; `findmnt /run/secrets` weist das Ziel als `tmpfs` aus. Gilt für beide Varianten.
- **Hauptweg:** `pct exec <vmid> -- stat -c '%a %U:%G' /var/lib/sops-nix/key.txt` zeigt `600 root:root` – fehlt die Datei, wurde Punkt 3 übersprungen oder die Variante ist aktiv.
- `pct exec <vmid> -- grep -rl "REGISTRY_MIRROR_PASSWORD" /run/current-system` liefert **keinen** Treffer – anders als `RUNNER_SITE`, das bewusst im Store bleiben darf (Klartextdatei, unverändert referenziert).
- `pct exec <vmid> -- systemctl cat 'gitea-runner-*'` zeigt zwei `EnvironmentFile=`-Zeilen: `${./runner.env}` unverändert aus Schritt 12 und `/run/secrets/runner-env` neu daneben.
- Der Runner-Dienst bleibt aktiv; `journalctl` zeigt keinen neuen Fehler.
- **Zweiter Fall:** `pct exec <vmid> -- grep -rl SSSD_LDAP_DEFAULT_AUTHTOK /nix/store` findet nur die unsubstituierte `sssd.conf` mit dem Platzhalter. `pct exec <vmid> -- grep ldap_default_authtok /var/lib/sssd/sssd.conf` zeigt dagegen den echten Wert, `stat -c '%a'` liefert `600` – geschrieben von `preStart` unter `umask 0177`, außerhalb des Stores.
- Login aus Schritt 8 funktioniert unverändert weiter.
- **Dritter Fall:** `pct exec <vmid> -- test -e /etc/gitea-runner-<runner-name>-token.env` schlägt fehl (Datei entfällt), `pct exec <vmid> -- cat /run/secrets/runner-token` zeigt das Token; der Runner bleibt in Forgejo online wie in Schritt 13.

## Wenn's schiefgeht

**`Failed to get the data key required to decrypt the SOPS file.`** – wie in Kapitel 10: Im Hauptweg zeigt `keyFile` auf die falsche/fehlende Datei, oder `<age-recipient>` fehlt in `.sops.yaml`. In der Variante zeigt `sshKeyPaths` auf den falschen Host-Key, oder dessen `ssh-to-age`-Ausgabe fehlt als Empfänger. Gegenprüfen, dann `sops updatekeys secrets/runner.env`.

**Build bricht mit *"No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home"* ab:** Nur im Hauptweg möglich – `sshKeyPaths = [ ]` übernommen, `keyFile` vergessen. Ohne Empfänger verweigert das Modul den Build per Assertion.<sup>3</sup> Fix: `keyFile` ergänzen.

**Build bricht mit `error: undefined variable 'config'` ab:** Funktionskopf nicht um `config` erweitert – nötig in beiden Varianten wegen `config.sops.secrets."runner-env".path`. Fix: Kopfzeile wie im Diff korrigieren.

**Build bricht mit `error: undefined variable 'SSSD_LDAP_DEFAULT_AUTHTOK'` ab:** `${SSSD_LDAP_DEFAULT_AUTHTOK}` statt `$SSSD_LDAP_DEFAULT_AUTHTOK` geschrieben – aus Shell-Gewohnheit naheliegend, aber `${...}` ist in `''…''` Nix-Interpolation, keine Shell-Syntax; Nix wertet den Namen als eigene Variable aus und scheitert, bevor `envsubst` läuft. Fix: bare `$VAR` wie im Diff.<sup>5</sup>

**Login scheitert ohne aussagekräftige Meldung, `journalctl -u sssd` zeigt nichts Auffälliges:** `secrets/sssd.env` fehlt eine Variable oder ihr Wert ist leer – `envsubst` ersetzt sie stillschweigend durch nichts, sssd bindet mit leerem Passwort, der Server lehnt ab, ohne das als Konfigurationsfehler zu melden. Gegenprüfen mit `sops secrets/sssd.env` auf fehlende/leere Zeilen.

## Rückweg

`EnvironmentFile`-Liste zurück auf `[ "${./runner.env}" ]`, `tokenFile` zurück auf `"/etc/gitea-runner-<runner-name>-token.env"` (Datei wie in Schritt 13 neu anlegen), `sops.*`-Zeilen und `config`-Argument entfernen, `sops-nix.nixosModules.sops` aus `flake.nix` streichen, rebuilden. Hauptweg zusätzlich: `pct exec <vmid> -- rm -rf /var/lib/sops-nix`. `secrets/*.env`/`.sops.yaml` können gefahrlos liegen bleiben. Zweiter Fall zusätzlich: `environmentFile`-/`sops.secrets."sssd-env"`-Zeilen aus `ldap.nix` entfernen, Bind-Variablen auf Klartext zurücksetzen (oder streichen bei anonymem Bind).

## Querverweis

sops-nix: Kapitel 10. SSH-Host-Key: Schritt 5. `pct push`: Schritt 2. Klartext-Gegenstück: Schritt 12, Token-Datei: Schritt 13, Registry-Spiegel: Schritt 14. LDAP-Bind-Warnbox: Schritt 8. Projekt 2/3 übernehmen die Variante über mehrere Hosts hinweg.

---

<sup>1</sup> Nix-Pfad-Literale vs. Strings: Teil I, Kapitel 3/2. `/run/secrets/…` als `tmpfs`-Ziel: [sops-nix – GitHub](https://github.com/Mic92/sops-nix), bereits Kapitel 10 zitiert.

<sup>2</sup> Quelle: sops-nix, `pkgs/sops-install-secrets/main.go`, `decryptSecret`: `case Binary, Dotenv, Ini:` liefert immer die ganze Datei, nur `Yaml`/`JSON` gehen über `recurseSecretKey`; `validateSopsFile` prüft den Schlüssel nur, wenn `Format != Binary && != Dotenv && != Ini`. https://github.com/Mic92/sops-nix/blob/master/pkgs/sops-install-secrets/main.go

<sup>3</sup> Quelle: sops-nix, `modules/sops/default.nix`. `age.keyFile`: `type = lib.types.nullOr pathNotInStore;`. `age.generateKey`: `default = false;`, "the key must already be present at the specified location." `age.sshKeyPaths`-Default: ed25519-Keys aus `config.services.openssh.hostKeys`. Assertion: "No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home". https://github.com/Mic92/sops-nix/blob/master/modules/sops/default.nix

<sup>4</sup> Quelle: `sops`-Quellcode (nicht sops-nix), `cmd/sops/main.go`, Befehl `updatekeys`: "update the keys of SOPS files using the config file". https://github.com/getsops/sops/blob/main/cmd/sops/main.go

<sup>5</sup> Quelle: nixpkgs, `nixos/modules/services/misc/sssd.nix`. `environmentFile`-Options-Beschreibung wörtlich: "Secrets may be passed to the service without adding them to the world-readable Nix store, by specifying placeholder variables as the option value in Nix and setting these variables accordingly in the environment file", Beispiel `ldap_default_authtok = $SSSD_LDAP_DEFAULT_AUTHTOK`. Unit-Kommentar: "We cannot use LoadCredential here because it's not available in ExecStartPre". https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/misc/sssd.nix

<sup>6</sup> Quelle: sops-nix, `modules/sops/default.nix`. `sops.secrets.<name>.restartUnits`: "works the same way as systemd.services.<name>.restartTriggers". `systemd.services.sops-install-secrets`: `wantedBy = [ "sysinit.target" ]`, zusätzlich ein `system.activationScripts`-Eintrag beim `switch`. https://github.com/Mic92/sops-nix/blob/master/modules/sops/default.nix

<sup>7</sup> Quelle: `systemd.exec`(5), `EnvironmentFile=`: "If the same variable is set twice from these files, the files will be read in the order they are specified and the later setting will override the earlier setting." https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html
