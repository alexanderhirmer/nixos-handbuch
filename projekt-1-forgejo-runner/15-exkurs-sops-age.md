---
title: "Exkurs: sops-age"
weight: 15
---

# Schritt 15: Exkurs — dieselbe `.env` sops-age-verschlüsselt, mit dem Schlüssel, den du schon hast

## Ziel

`<repo-root>/secrets/runner.env` liegt sops-age-verschlüsselt neben dem Klartext aus Schritt 12: im Hauptweg entschlüsselt durch deinen age-Schlüssel, in einer Variante durch einen aus dem SSH-Host-Key abgeleiteten Zweitschlüssel; `forgejo-runner.nix` zeigt am Ende auf `/run/secrets/…`.

## Voraussetzung

Teil I, Kapitel 10 ist gelesen – sops-nix, age vs. GPG, `/run/secrets/…` werden vorausgesetzt. Du besitzt ein age-Schlüsselpaar (`<age-recipient>`, `<age-key-file>`) – dieser Exkurs erzeugt keins, weder per `age-keygen` noch über `sops.age.generateKey`. Schritt 5 (SSH-Host-Key, nur Variante) und 12/13 sind abgeschlossen: `forgejo-runner.nix` hat seit Schritt 12 `{ pkgs, utils, ... }:` im Kopf und `EnvironmentFile = [ "${./runner.env}" ];`, gemergt mit `tokenFile` aus Schritt 11.

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

## Prüfen

- `pct exec <vmid> -- cat /run/secrets/runner-env` zeigt denselben Klartext wie beim `sops`-Aufruf; `findmnt /run/secrets` weist das Ziel als `tmpfs` aus. Gilt für beide Varianten.
- **Hauptweg:** `pct exec <vmid> -- stat -c '%a %U:%G' /var/lib/sops-nix/key.txt` zeigt `600 root:root` – fehlt die Datei, wurde Punkt 3 übersprungen oder die Variante ist aktiv.
- `pct exec <vmid> -- grep -rl "RUNNER_SITE" /run/current-system` liefert **keinen** Treffer mehr: Nur der Pfad wird referenziert; der alte Store-Pfad bleibt bis `nix-collect-garbage` liegen – die neue Fassung landet dort **nie**.
- `pct exec <vmid> -- systemctl cat 'gitea-runner-*'` zeigt die `EnvironmentFile=`-Zeile aus Schritt 13 unverändert sowie eine zweite, jetzt auf `/run/secrets/runner-env`.
- Der Runner-Dienst bleibt aktiv; `journalctl` zeigt keinen neuen Fehler.

## Wenn's schiefgeht

**`Failed to get the data key required to decrypt the SOPS file.`** – wie in Kapitel 10: Im Hauptweg zeigt `keyFile` auf die falsche/fehlende Datei, oder `<age-recipient>` fehlt in `.sops.yaml`. In der Variante zeigt `sshKeyPaths` auf den falschen Host-Key, oder dessen `ssh-to-age`-Ausgabe fehlt als Empfänger. Gegenprüfen, dann `sops updatekeys secrets/runner.env`.

**Build bricht mit *"No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home"* ab:** Nur im Hauptweg möglich – `sshKeyPaths = [ ]` übernommen, `keyFile` vergessen. Ohne Empfänger verweigert das Modul den Build per Assertion.<sup>3</sup> Fix: `keyFile` ergänzen.

**Build bricht mit `error: undefined variable 'config'` ab:** Funktionskopf nicht um `config` erweitert – nötig in beiden Varianten wegen `config.sops.secrets."runner-env".path`. Fix: Kopfzeile wie im Diff korrigieren.

## Rückweg

`EnvironmentFile`-Liste zurück auf `[ "${./runner.env}" ]`, `sops.*`-Zeilen und `config`-Argument entfernen, `sops-nix.nixosModules.sops` aus `flake.nix` streichen, rebuilden. Im Hauptweg zusätzlich `pct exec <vmid> -- rm -rf /var/lib/sops-nix`. `secrets/runner.env`/`.sops.yaml` können gefahrlos im Repo bleiben.

## Querverweis

sops-nix: Kapitel 10. SSH-Host-Key: Schritt 5. `pct push`: Schritt 2. Klartext-Gegenstück: Schritt 12, Token-Datei: Schritt 13. Projekt 2/3 übernehmen die Variante über mehrere Hosts hinweg.

---

<sup>1</sup> Nix-Pfad-Literale vs. Strings: Teil I, Kapitel 3/2. `/run/secrets/…` als `tmpfs`-Ziel: [sops-nix – GitHub](https://github.com/Mic92/sops-nix), bereits Kapitel 10 zitiert.

<sup>2</sup> Quelle: sops-nix, `pkgs/sops-install-secrets/main.go`, `decryptSecret`: `case Binary, Dotenv, Ini:` liefert immer die ganze Datei, nur `Yaml`/`JSON` gehen über `recurseSecretKey`; `validateSopsFile` prüft den Schlüssel nur, wenn `Format != Binary && != Dotenv && != Ini`. https://github.com/Mic92/sops-nix/blob/master/pkgs/sops-install-secrets/main.go

<sup>3</sup> Quelle: sops-nix, `modules/sops/default.nix`. `age.keyFile`: `type = lib.types.nullOr pathNotInStore;`. `age.generateKey`: `default = false;`, "the key must already be present at the specified location." `age.sshKeyPaths`-Default: ed25519-Keys aus `config.services.openssh.hostKeys`. Assertion: "No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home". https://github.com/Mic92/sops-nix/blob/master/modules/sops/default.nix

<sup>4</sup> Quelle: `sops`-Quellcode (nicht sops-nix), `cmd/sops/main.go`, Befehl `updatekeys`: "update the keys of SOPS files using the config file". https://github.com/getsops/sops/blob/main/cmd/sops/main.go
