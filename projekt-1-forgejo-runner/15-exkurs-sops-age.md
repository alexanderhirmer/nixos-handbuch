---
title: "Exkurs: sops-age"
weight: 15
---

# Schritt 15: Exkurs — dieselbe `.env` liegt sops-age-verschlüsselt vor, mit dem Schlüssel, den du schon hast

## Ziel

Neben `modules/runner/runner.env` (Schritt 12, Klartext im Store) existiert `<repo-root>/secrets/runner.env` (sops-age-verschlüsselt) plus `<repo-root>/.sops.yaml`. Im **Hauptweg** ist dein bereits vorhandener age-Schlüssel (`<age-recipient>`/`<age-key-file>`, Platzhaltertabelle in `00-uebersicht.md`) der einzige Empfänger und liegt zusätzlich auf dem Container, wo sops-nix ihn über `sops.age.keyFile` einliest. Eine **Variante** am Ende ersetzt das durch eine aus dem SSH-Host-Key des Containers abgeleitete Identität als zweiten Empfänger. In beiden Fällen zeigt `forgejo-runner.nix` am Ende auf das verschlüsselte Secret, das zur Laufzeit unter `/run/secrets/…` liegt – nie im Nix-Store.

## Voraussetzung

Teil I, Kapitel 10 ("Secrets") ist gelesen – sops-nix-Grundmechanik, age vs. GPG und `/run/secrets/…` werden hier vorausgesetzt, nicht wiederholt. Du besitzt bereits ein age-Schlüsselpaar: `<age-recipient>` als öffentlichen Empfänger (beginnt mit `age1`) und `<age-key-file>` als private Schlüsseldatei auf deiner Workstation – dieser Exkurs erzeugt an keiner Stelle einen neuen Schlüssel, weder per `age-keygen` noch über `sops.age.generateKey`. Schritt 5 (Container hat einen SSH-Host-Key, standardmäßig auch als ed25519 unter `/etc/ssh/ssh_host_ed25519_key` – erst für die Variante relevant) und Schritt 12/13 sind abgeschlossen: `forgejo-runner.nix` trägt seit Schritt 12 `{ pkgs, utils, ... }:` im Funktionskopf und setzt `systemd.services."gitea-runner-${utils.escapeSystemdPath "<runner-name>"}".serviceConfig.EnvironmentFile = [ "${./runner.env}" ];` – eine zweite, mit der `tokenFile`-Zeile aus Schritt 11 zusammengeführte `EnvironmentFile`-Angabe (Listen-Merge von `unitOption`, siehe Schritt 12).

## Durchführung

**1. sops-nix als Flake-Input.** `github:Mic92/sops-nix` hat keine versionierten Release-Tags (`git ls-remote --tags` zeigt nur einen unrelated `assets`-Tag) – genau wie bei `nixpkgs` in Schritt 2 ist deshalb nicht der Branchname der eigentliche Pin, sondern `flake.lock`, das den exakten Commit festhält und mitversioniert wird.

**2. `.sops.yaml` mit `<age-recipient>` als Empfänger anlegen**, `secrets/runner.env` verschlüsseln (dotenv-Format explizit, statt auf Endungs-Erkennung zu vertrauen):

```console
$ sops --input-type dotenv --output-type dotenv secrets/runner.env
```

**3. Den privaten Schlüssel auf den Container bringen.** Das ist der heikle Teil: `<age-key-file>` liegt auf deiner Workstation und muss auf den Container, ohne dabei je im Repo oder im Store zu landen. Muster wie Schritt 2 (`pct push`) und Schritt 13 (Zielverzeichnis anlegen, `chown`, `chmod`), diesmal vom Proxmox-Host aus:

```console
$ pct exec <vmid> -- mkdir -p /var/lib/sops-nix
$ pct push <vmid> <age-key-file> /var/lib/sops-nix/key.txt
$ pct exec <vmid> -- chown root:root /var/lib/sops-nix/key.txt
$ pct exec <vmid> -- chmod 600 /var/lib/sops-nix/key.txt
```

Der Schlüssel gehört an keiner Stelle ins Repo – auch nicht verschlüsselt, das ist ja gerade der Unterschied zum öffentlichen Empfänger in `.sops.yaml` – und erst recht nicht als Nix-Pfad-Literal wie `./key.txt` referenziert: Das würde ihn beim nächsten Build direkt in den weltlesbaren Store kopieren, exakt das Gegenteil von Geheimhaltung. Das ist hier kein Stilhinweis, den man ignorieren könnte – `sops.age.keyFile` ist im sops-nix-Quellcode als `lib.types.nullOr pathNotInStore` typisiert;<sup>3</sup> ein Wert unterhalb von `/nix/store` wird vom Modulsystem zur Auswertungszeit aktiv abgelehnt, das System baut in diesem Fall gar nicht erst.

**4. `forgejo-runner.nix` erweitern**, pushen, rebuilden (Diff siehe "Dateien").

**Variante — getrennte Rollen (kompakt).** Statt den privaten Schlüssel zu verteilen, bleibt er auf der Workstation; der Container entschlüsselt über eine aus seinem eigenen SSH-Host-Key abgeleitete Identität:

```console
$ pct exec <vmid> -- cat /etc/ssh/ssh_host_ed25519_key.pub | \
    nix shell nixpkgs#ssh-to-age -c ssh-to-age
```

Die Ausgabe kommt als zweiter Empfänger neben `<age-recipient>` in `.sops.yaml` (Diff siehe "Dateien"), danach müssen bereits verschlüsselte Dateien für den neuen Empfänger nachgezogen werden:<sup>4</sup>

```console
$ sops updatekeys secrets/runner.env
```

`forgejo-runner.nix` setzt in dieser Variante `sops.age.sshKeyPaths` statt `keyFile`/`generateKey` (Diff siehe "Dateien") – der private Schlüssel selbst verlässt die Workstation nie.

**Die Abwägung dahinter.** Der Hauptweg hat genau einen Empfänger, verlangt nach einem Neuaufsetzen des Containers kein `sops updatekeys` und ist konzeptionell am einfachsten – der Preis ist, dass der private Schlüssel auf jedem Zielsystem liegt, das ihn nutzt: Wer dort Root hat, hat den Schlüssel, und damit Zugriff auf *alle* Secrets, die je damit verschlüsselt wurden, nicht nur die dieses Containers. Die Variante zahlt dafür mit mehr laufender Pflege – zwei Empfänger in `.sops.yaml`, ein zusätzlicher `sops updatekeys`-Lauf nach jedem Neuaufsetzen eines Hosts –, aber der private Schlüssel verlässt die Workstation nie, und ein kompromittierter Container gibt nur seine eigene, host-gebundene Identität preis, nicht den Generalschlüssel. Für einen einzelnen Host wie in diesem Projekt ist der Hauptweg vertretbar; sobald mehrere Hosts denselben Empfänger teilen würden, kippt die Abwägung – Projekt 2 und 3 gehen deshalb den Weg der Variante.

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

`<repo-root>/secrets/runner.env` (neu, verschlüsselt – der Klartext, den du beim `sops`-Aufruf oben in den Editor tippst, ist derselbe wie in Schritt 12):

```bash
RUNNER_ENVIRONMENT=produktion
RUNNER_SITE=rz-intern
TZ=Europe/Berlin
```

Im Repo liegt davon nur die verschlüsselte Fassung. Das `dotenv`-Format verschlüsselt dabei ausschließlich die *Werte* – die Variablennamen bleiben im Klartext lesbar, wodurch ein `git diff` weiterhin zeigt, *welcher* Eintrag sich geändert hat, ohne dessen Inhalt preiszugeben. Bei `format = "binary"` wäre die ganze Datei ein undurchsichtiger Block; möglich wäre auch das, es kostet nur genau diese Lesbarkeit.

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

- **`keyFile = "/var/lib/sops-nix/key.txt";`** – der Pfad, unter dem Punkt 3 oben den privaten Schlüssel abgelegt hat. Ein String, kein Pfad-Literal (siehe Punkt 3) – genau wie `tokenFile` in Schritt 13.
- **`generateKey = false;`** – ist bereits der Default des Moduls ("Whether or not to generate the age key. If this option is set to false, the key must already be present at the specified location."<sup>3</sup>) und ändert damit an sich nichts. Explizit hingeschrieben macht die Zeile aber unmissverständlich, was dieser Container *nicht* tut: nie selbst einen Schlüssel erzeugen, immer einen fertigen erwarten – exakt die Vorgabe, unter der dieses ganze Projekt steht.
- **`sshKeyPaths = [ ];`** – muss explizit leer sein. Der Default sind die ed25519-Keys aus `config.services.openssh.hostKeys`;<sup>3</sup> ohne diese Zeile würde sops-nix den ed25519-Host-Key des Containers automatisch *zusätzlich* zu `keyFile` als zweite Identität einhängen. Dann wäre "nur mein Schlüssel zählt" schlicht falsch – der Container könnte über seinen eigenen Host-Key entschlüsseln, sobald dieser irgendwann als Empfänger in `.sops.yaml` landet, ohne dass das an dieser Stelle sichtbar wäre.

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

`./runner.env` ist ein Nix-**Pfad**-Literal – wird beim Bauen in den Store kopiert (so funktioniert Schritt 12 überhaupt erst). `config.sops.secrets."runner-env".path` ist dagegen zur Auswertungszeit nur ein **String**, der erst zur Aktivierungszeit auf `/run/secrets/runner-env` zeigt; dort landet der Klartext nie im Store, sondern in einem `tmpfs` außerhalb davon.<sup>1</sup> `flake.nix` ändert sich in der Variante unten nicht erneut.

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

`<repo-root>/modules/runner/forgejo-runner.nix` (geändert, gegenüber der Hauptweg-Fassung oben – nur der `sops.age`-Block wechselt, der Rest der Datei bleibt identisch):

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

Kein `keyFile`, kein privater Schlüssel auf dem Container – `sshKeyPaths` genügt, weil der Host-Key ohnehin schon dort liegt (Schritt 5).

## Prüfen

- `pct exec <vmid> -- cat /run/secrets/runner-env` zeigt denselben Klartext, der beim `sops`-Aufruf eingegeben wurde – und `findmnt /run/secrets` weist das Ziel als `ramfs`/`tmpfs` aus, nicht als Teil des Stores. Gilt unabhängig von der gewählten Variante.
- **Hauptweg:** `pct exec <vmid> -- stat -c '%a %U:%G' /var/lib/sops-nix/key.txt` zeigt `600 root:root`. Existiert diese Datei nicht, wurde entweder Punkt 3 übersprungen oder die Variante ist aktiv – dort gibt es sie bewusst nicht.
- `pct exec <vmid> -- grep -rl "RUNNER_SITE" /run/current-system` liefert **keinen** Treffer mehr: Im aktiven System wird nur noch der Pfad `/run/secrets/runner-env` referenziert, nicht der Inhalt. Der *alte* Store-Pfad aus Schritt 12 liegt weiterhin unter `/nix/store` und ist dort auch weiterhin lesbar – er ist nur nicht mehr referenziert und verschwindet erst mit `nix-collect-garbage`. Der Punkt des Exkurses ist, dass die neue Fassung dort **nie** ankommt, nicht dass die alte rückwirkend verschwindet.
- `pct exec <vmid> -- systemctl cat 'gitea-runner-*'` zeigt im `[Service]`-Abschnitt die `EnvironmentFile=`-Zeile aus Schritt 13 unverändert (`/etc/gitea-runner-<runner-name>-token.env`) sowie eine zweite, die jetzt auf `/run/secrets/runner-env` zeigt statt auf einen `/nix/store/…`-Pfad.
- Der Runner-Dienst (Schritt 13) bleibt aktiv; `journalctl` zeigt keinen neuen Fehler nach der Umstellung.

## Wenn's schiefgeht

**`Failed to get the data key required to decrypt the SOPS file.`** – dieselbe Meldung wie in Kapitel 10, mit unterschiedlicher Ursache je Variante: Im Hauptweg zeigt `sops.age.keyFile` auf die falsche oder eine nicht vorhandene Datei, oder `<age-recipient>` fehlt in `.sops.yaml`. In der Variante zeigt `sops.age.sshKeyPaths` auf den falschen Host-Key, oder dessen `ssh-to-age`-Ausgabe fehlt als Empfänger. Pfad bzw. Empfängerliste gegenprüfen und nach einer Korrektur `sops updatekeys secrets/runner.env` laufen lassen.

**Build bricht mit *"No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home"* ab:** Nur im Hauptweg möglich – `sshKeyPaths = [ ]` wurde übernommen, aber `keyFile` beim Kopieren des Diffs vergessen. Ohne `keyFile` bleibt kein Empfänger übrig, aus dem sops-nix eine Identität ableiten könnte, und das Modul verweigert den Build über eine Assertion, statt später mit einer unklaren Laufzeitmeldung zu scheitern.<sup>3</sup> Fix: `keyFile` wie im Diff ergänzen.

**Build bricht mit `error: undefined variable 'config'` ab:** Der Funktionskopf von `forgejo-runner.nix` wurde nicht um `config` erweitert (Schritt 12 hatte dort bereits `utils` ergänzt, jetzt kommt `config` hinzu) – `...` allein bindet keine benannten Modulargumente. Fix: Kopfzeile wie im Diff korrigieren. Gilt für beide Varianten, da beide `config.sops.secrets."runner-env".path` referenzieren.

## Rückweg

Die `EnvironmentFile`-Liste in `forgejo-runner.nix` zurück auf `[ "${./runner.env}" ]` setzen, die `sops.*`-Zeilen und das ergänzte `config`-Argument entfernen, `sops-nix.nixosModules.sops` aus `flake.nix` streichen, rebuilden. Im Hauptweg zusätzlich den privaten Schlüssel vom Container entfernen: `pct exec <vmid> -- rm -rf /var/lib/sops-nix` – er hat dort nach dem Rückbau keine Aufgabe mehr. `secrets/runner.env` und `.sops.yaml` können in jeder Variante gefahrlos im Repo bleiben – verschlüsselt, ohne Store-Bezug, unabhängig davon, welcher Weg zuletzt aktiv war.

## Querverweis

sops-nix-Grundlagen: Teil I, Kapitel 10. SSH-Host-Key: Schritt 5. `pct push`-Muster: Schritt 2. Klartext-Gegenstück: Schritt 12; Token-Datei nach demselben "außerhalb des Stores"-Prinzip: Schritt 13. Projekt 2 und 3 übernehmen die Variante mit getrennten Rollen vollständig — dort für echte Zugangsdaten (DB-Passwort, Vaultwarden-Admin-Token) mit statischen Dienst-Nutzern statt `DynamicUser`, und über mehr als einen Host hinweg, wo der Hauptweg dieses Exkurses nicht mehr trägt.

---

<sup>1</sup> Nix-Pfad-Literale vs. Strings und Store-Kopie: Teil I, Kapitel 3 ("Nix als Sprache") und Kapitel 2 ("Das Nix-Modell"). `/run/secrets/…` als `tmpfs`-Ziel von sops-nix: [sops-nix – GitHub](https://github.com/Mic92/sops-nix), bereits in Kapitel 10 zitiert.

<sup>2</sup> Quelle: sops-nix-Quellcode. Options-Beschreibung in `modules/sops/default.nix` (`key`: Default `config._module.args.name`, "This option is ignored if format is binary. \"\" means whole file."), tatsächliche Umsetzung in `pkgs/sops-install-secrets/main.go`: In `decryptSecret` bekommt `case Binary, Dotenv, Ini:` immer `sourceFile.binary`, also die ganze Datei, während nur `Yaml`/`JSON` über `recurseSecretKey` gehen; `validateSopsFile` prüft den Schlüssel nur, wenn `s.Format != Binary && s.Format != Dotenv && s.Format != Ini`. https://github.com/Mic92/sops-nix/blob/master/pkgs/sops-install-secrets/main.go

<sup>3</sup> Quelle: sops-nix-Quellcode, `modules/sops/default.nix`. `sops.age.keyFile`: `type = lib.types.nullOr pathNotInStore;`, `default = null;`, `example = "/var/lib/sops-nix/key.txt";`, Beschreibung "Path to age key file used for sops decryption." `sops.age.generateKey`: `default = false;`, Beschreibung "Whether or not to generate the age key. If this option is set to false, the key must already be present at the specified location." `sops.age.sshKeyPaths`: Default sind die ed25519-Keys aus `config.services.openssh.hostKeys`. Assertion bei fehlender Schlüsselquelle, Wortlaut: "No key source configured for sops. Either set services.openssh.enable or set sops.age.keyFile or sops.gnupg.home". https://github.com/Mic92/sops-nix/blob/master/modules/sops/default.nix

<sup>4</sup> Quelle: `sops`-Quellcode (nicht sops-nix), `cmd/sops/main.go`: CLI-Befehl `updatekeys`, Usage "update the keys of SOPS files using the config file" – liest `.sops.yaml` neu und verschlüsselt den Data Key für hinzugekommene bzw. entfernte Empfänger neu, ohne den restlichen Dateiinhalt anzufassen. https://github.com/getsops/sops/blob/main/cmd/sops/main.go
