---
title: "Exkurs: sops-age"
weight: 15
---

# Schritt 15: Exkurs — dieselbe `.env` liegt sops-age-verschlüsselt vor

## Ziel

Neben `modules/runner/runner.env` (Schritt 12, Klartext im Store) existiert `<repo-root>/secrets/runner.env` (sops-age-verschlüsselt) plus `<repo-root>/.sops.yaml`. Umgestellt zeigt `forgejo-runner.nix` auf das verschlüsselte Secret, das zur Laufzeit unter `/run/secrets/…` liegt – nie im Nix-Store.

## Voraussetzung

Teil I, Kapitel 10 ("Secrets") ist gelesen – sops-nix-Grundmechanik, age vs. GPG und `/run/secrets/…` werden hier vorausgesetzt, nicht wiederholt. Schritt 5 (Container hat einen SSH-Host-Key, standardmäßig auch als ed25519 unter `/etc/ssh/ssh_host_ed25519_key`) und Schritt 12/13 sind abgeschlossen: `forgejo-runner.nix` trägt seit Schritt 12 `{ pkgs, utils, ... }:` im Funktionskopf und setzt `systemd.services."gitea-runner-${utils.escapeSystemdPath "<runner-name>"}".serviceConfig.EnvironmentFile = [ "${./runner.env}" ];` – eine zweite, mit der `tokenFile`-Zeile aus Schritt 11 zusammengeführte `EnvironmentFile`-Angabe (Listen-Merge von `unitOption`, siehe Schritt 12).

## Durchführung

**1. sops-nix als Flake-Input.** `github:Mic92/sops-nix` hat keine versionierten Release-Tags (`git ls-remote --tags` zeigt nur einen unrelated `assets`-Tag) – genau wie bei `nixpkgs` in Schritt 2 ist deshalb nicht der Branchname der eigentliche Pin, sondern `flake.lock`, das den exakten Commit festhält und mitversioniert wird.

**2. Age-Schlüssel aus dem vorhandenen SSH-Host-Key ableiten** – anders als Kapitel 10 (dort: frischer `age-keygen`-Schlüssel) wird hier der **bereits vorhandene** Host-Key des Containers wiederverwendet, kein separates Schlüsselmaterial:

```console
$ pct exec <vmid> -- cat /etc/ssh/ssh_host_ed25519_key.pub | \
    nix shell nixpkgs#ssh-to-age -c ssh-to-age
```

**3. `.sops.yaml` mit diesem Empfänger anlegen**, `secrets/runner.env` verschlüsseln (dotenv-Format explizit, statt auf Endungs-Erkennung zu vertrauen):

```console
$ sops --input-type dotenv --output-type dotenv secrets/runner.env
```

**4. `forgejo-runner.nix` erweitern**, pushen, rebuilden (siehe "Dateien").

## Dateien

`<repo-root>/.sops.yaml` (neu):

```yaml
# <repo-root>/.sops.yaml
creation_rules:
  - path_regex: secrets/.*\.env$
    key_groups:
      - age:
          # Hier die tatsaechliche ssh-to-age-Ausgabe aus Punkt 2 einsetzen.
          # Ein age-Empfaenger beginnt mit "age1"; ein Beispielwert waere
          # hier gefaehrlich, weil er sich kommentarlos kopieren liesse.
          - age1...
```

`<repo-root>/secrets/runner.env` (neu, verschlüsselt – der Klartext, den du beim `sops`-Aufruf oben in den Editor tippst, ist derselbe wie in Schritt 12):

```bash
RUNNER_ENVIRONMENT=produktion
RUNNER_SITE=rz-intern
TZ=Europe/Berlin
```

Im Repo liegt davon nur die verschlüsselte Fassung. Das `dotenv`-Format verschlüsselt dabei ausschließlich die *Werte* – die Variablennamen bleiben im Klartext lesbar, wodurch ein `git diff` weiterhin zeigt, *welcher* Eintrag sich geändert hat, ohne dessen Inhalt preiszugeben. Bei `format = "binary"` wäre die ganze Datei ein undurchsichtiger Block; möglich wäre auch das, es kostet nur genau diese Lesbarkeit.

> 💡 **Nice to know:** Die Options-Beschreibung von `sops.secrets.<name>.key` legt eine Falle aus: Sie sagt, der Schlüssel werde "in der sops-Datei nachgeschlagen", der Default sei der Name des Secrets, und `""` bedeute "whole file" – man könnte also meinen, hier müsse zwingend `key = "";` stehen, sonst suche sops-nix einen Eintrag namens `runner-env` *innerhalb* der Datei. Für `dotenv` stimmt das nicht: Im Go-Quellcode landet `dotenv` – zusammen mit `binary` und `ini` – in dem Zweig, der immer den gesamten entschlüsselten Inhalt übernimmt, und die Schlüsselprüfung überspringt diese drei Formate ausdrücklich. `key` ist hier also wirkungslos, nicht Pflicht. Nachgeschlagen wird nur bei `yaml` und `json`.<sup>2</sup>

`<repo-root>/modules/runner/forgejo-runner.nix` (geändert):

```diff
--- a/modules/runner/forgejo-runner.nix
+++ b/modules/runner/forgejo-runner.nix
@@
-{ pkgs, utils, ... }:
+{ config, pkgs, utils, ... }:
 {
+  sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
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

`sops.secrets."runner-env"` braucht hier **keine** eigene `mode`/`owner`-Einstellung – der sops-nix-Default (`root:root`, `0400`) genügt. Der Grund ist derselbe wie bei `tokenFile` in Schritt 13: `EnvironmentFile=` liest der systemd-**Manager** (root), bevor er auf den `DynamicUser`-Nutzer `gitea-runner` wechselt – der Runner-Prozess selbst braucht nie Leserechte auf diese Datei.

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

`./runner.env` ist ein Nix-**Pfad**-Literal – wird beim Bauen in den Store kopiert (so funktioniert Schritt 12 überhaupt erst). `config.sops.secrets."runner-env".path` ist dagegen zur Auswertungszeit nur ein **String**, der erst zur Aktivierungszeit auf `/run/secrets/runner-env` zeigt; dort landet der Klartext nie im Store, sondern in einem `tmpfs` außerhalb davon.<sup>1</sup>

## Prüfen

- `pct exec <vmid> -- cat /run/secrets/runner-env` zeigt denselben Klartext, der beim `sops`-Aufruf eingegeben wurde – und `findmnt /run/secrets` weist das Ziel als `ramfs`/`tmpfs` aus, nicht als Teil des Stores.
- `pct exec <vmid> -- grep -rl "RUNNER_SITE" /run/current-system` liefert **keinen** Treffer mehr: Im aktiven System wird nur noch der Pfad `/run/secrets/runner-env` referenziert, nicht der Inhalt. Ehrlicherweise: Der *alte* Store-Pfad aus Schritt 12 liegt weiterhin unter `/nix/store` und ist dort auch weiterhin lesbar – er ist nur nicht mehr referenziert und verschwindet erst mit `nix-collect-garbage`. Der Punkt des Exkurses ist, dass die neue Fassung dort **nie** ankommt, nicht dass die alte rückwirkend verschwindet.
- `pct exec <vmid> -- systemctl cat 'gitea-runner-*'` zeigt im `[Service]`-Abschnitt die `EnvironmentFile=`-Zeile aus Schritt 13 unverändert (`/etc/gitea-runner-<runner-name>-token.env`) sowie eine zweite, die jetzt auf `/run/secrets/runner-env` zeigt statt auf einen `/nix/store/…`-Pfad.
- Der Runner-Dienst (Schritt 13) bleibt aktiv; `journalctl` zeigt keinen neuen Fehler nach der Umstellung.

## Wenn's schiefgeht

**`Failed to get the data key required to decrypt the SOPS file.`** – dieselbe Meldung wie in Kapitel 10: `sops.age.sshKeyPaths` zeigt auf den falschen Host-Key, oder dessen Public Key fehlt in `.sops.yaml`. Mit dem Befehl aus Schritt 2 gegenprüfen und `sops updatekeys secrets/runner.env` nach einer Korrektur.

**Build bricht mit `error: undefined variable 'config'` ab:** Der Funktionskopf von `forgejo-runner.nix` wurde nicht um `config` erweitert (Schritt 12 hatte dort bereits `utils` ergänzt, jetzt kommt `config` hinzu) – `...` allein bindet keine benannten Modulargumente. Fix: Kopfzeile wie im Diff korrigieren.

## Rückweg

Die `EnvironmentFile`-Liste in `forgejo-runner.nix` zurück auf `[ "${./runner.env}" ]` setzen, die `sops.*`-Zeilen und das ergänzte `config`-Argument entfernen, `sops-nix.nixosModules.sops` aus `flake.nix` streichen, rebuilden. `secrets/runner.env` und `.sops.yaml` können gefahrlos im Repo bleiben – verschlüsselt, ohne Store-Bezug.

## Querverweis

sops-nix-Grundlagen: Teil I, Kapitel 10. SSH-Host-Key: Schritt 5. Klartext-Gegenstück: Schritt 12; Token-Datei nach demselben "außerhalb des Stores"-Prinzip: Schritt 13. Projekt 2 übernimmt sops-age vollständig – dort für echte Zugangsdaten (DB-Passwort, Vaultwarden-Admin-Token) mit statischen Dienst-Nutzern statt `DynamicUser`.

---

<sup>1</sup> Nix-Pfad-Literale vs. Strings und Store-Kopie: Teil I, Kapitel 3 ("Nix als Sprache") und Kapitel 2 ("Das Nix-Modell"). `/run/secrets/…` als `tmpfs`-Ziel von sops-nix: [sops-nix – GitHub](https://github.com/Mic92/sops-nix), bereits in Kapitel 10 zitiert.

<sup>2</sup> Quelle: sops-nix-Quellcode. Options-Beschreibung in `modules/sops/default.nix` (`key`: Default `config._module.args.name`, "This option is ignored if format is binary. \"\" means whole file."), tatsächliche Umsetzung in `pkgs/sops-install-secrets/main.go`: In `decryptSecret` bekommt `case Binary, Dotenv, Ini:` immer `sourceFile.binary`, also die ganze Datei, während nur `Yaml`/`JSON` über `recurseSecretKey` gehen; `validateSopsFile` prüft den Schlüssel nur, wenn `s.Format != Binary && s.Format != Dotenv && s.Format != Ini`. https://github.com/Mic92/sops-nix/blob/master/pkgs/sops-install-secrets/main.go
