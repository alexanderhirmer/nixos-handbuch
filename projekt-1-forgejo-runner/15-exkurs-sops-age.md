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
          - age1qyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqsyzhk3l
```

`<repo-root>/secrets/runner.env` (neu, verschlüsselt – Inhalt im Editor beim `sops`-Aufruf oben):

```
FORGEJO_RUNNER_LOG_LEVEL=info
```

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

- `pct exec <vmid> -- cat /run/secrets/runner-env` zeigt denselben Klartext, der beim `sops`-Aufruf eingegeben wurde.
- `pct exec <vmid> -- grep -r "FORGEJO_RUNNER_LOG_LEVEL" /nix/store` liefert **keinen** Treffer – Kontrast zu Schritt 12, wo `runner.env` im Store selbst durchsuchbar ist.
- Der Runner-Dienst (Schritt 13) bleibt aktiv; `journalctl` zeigt keinen neuen Fehler nach dem Umstellen von `env_file`.

## Wenn's schiefgeht

**`Failed to get the data key required to decrypt the SOPS file.`** – dieselbe Meldung wie in Kapitel 10: `sops.age.sshKeyPaths` zeigt auf den falschen Host-Key, oder dessen Public Key fehlt in `.sops.yaml`. Mit dem Befehl aus Schritt 2 gegenprüfen und `sops updatekeys secrets/runner.env` nach einer Korrektur.

**`/run/secrets/runner-env` existiert, `journalctl` zeigt trotzdem "Permission denied" beim Lesen:** `mode` wurde vergessen oder auf den (Default-)Wert `0400` zurückgesetzt – für den DynamicUser-Dienst nicht lesbar, siehe Kommentar oben.

## Rückweg

`env_file` in `forgejo-runner.nix` zurück auf `./runner.env` setzen, die `sops.*`-Zeilen entfernen, `sops-nix.nixosModules.sops` aus `flake.nix` streichen, rebuilden. `secrets/runner.env` und `.sops.yaml` können gefahrlos im Repo bleiben – verschlüsselt, ohne Store-Bezug.

## Querverweis

sops-nix-Grundlagen: Teil I, Kapitel 10. SSH-Host-Key: Schritt 5. Klartext-Gegenstück: Schritt 12; Token-Datei nach demselben "außerhalb des Stores"-Prinzip: Schritt 13. Projekt 2 übernimmt sops-age vollständig – dort für echte Zugangsdaten (DB-Passwort, Vaultwarden-Admin-Token) mit statischen Dienst-Nutzern statt `DynamicUser`.

---

<sup>1</sup> Nix-Pfad-Literale vs. Strings und Store-Kopie: Teil I, Kapitel 3 ("Nix als Sprache") und Kapitel 2 ("Das Nix-Modell"). `/run/secrets/…` als `tmpfs`-Ziel von sops-nix: [sops-nix – GitHub](https://github.com/Mic92/sops-nix), bereits in Kapitel 10 zitiert.
