---
title: "Runner-Konfiguration als .env"
weight: 12
---

# Schritt 12: Eine `.env`-Datei mit Klartext-Werten ist angelegt und eingebunden

## Ziel

`modules/runner/runner.env` enthält unkritische Konfigurationswerte für den Runner-Betrieb und wird dem in Schritt 11 erzeugten systemd-Dienst als zusätzliche `EnvironmentFile` mitgegeben – ohne die dort bereits gesetzte Token-Datei zu verdrängen.

## Voraussetzung

Schritt 11 ist abgeschlossen: `modules/runner/forgejo-runner.nix` definiert die Instanz `<runner-name>` mit `tokenFile`; der erzeugte Dienst existiert, ist aber weiterhin `failed`, weil diese Token-Datei noch fehlt (kommt in Schritt 13).

## Durchführung

**1. `runner.env` schreiben** (siehe "Dateien").

**2. `forgejo-runner.nix` erweitern** (Diff siehe "Dateien") und **3. testen, dann aktivieren:**

```console
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

**Warum das Runner-Modul selbst keine Option dafür hat:** `services.gitea-actions-runner` kennt pro Instanz nur `tokenFile` (speziell für das Token) und `settings` (YAML für den Runner-Daemon selbst, kein `.env`-Format) – keine generische Option für zusätzliche Umgebungsvariablen.<sup>1</sup> Der einzige saubere Weg ist deshalb `systemd.services.<unit>.serviceConfig.EnvironmentFile`, direkt am generischen systemd-Modul vorbei am Runner-Modul. Der reale Unit-Name ist – wie in Schritt 11 hergeleitet – `gitea-runner-${escapeSystemdPath "<runner-name>"}`, nicht `gitea-runner-<runner-name>` mit wörtlichen Bindestrichen.

**Die Falle dabei:** Schritt 11 hat über `instance.tokenFile` bereits intern `serviceConfig.EnvironmentFile = "/etc/gitea-runner-<runner-name>-token.env";` gesetzt – ein einzelner String. Setzt diese Datei dieselbe Option ein zweites Mal ebenfalls als String, wertet NixOS das als Konflikt. Der zugrundeliegende Options-Typ `unitOption` in `nixos/lib/systemd-unit-options.nix` merged dagegen automatisch zu einer Liste, sobald *irgendeine* der beteiligten Definitionen selbst eine Liste ist: `if any (def: isList def.value) defs then concatMap (def: toList def.value) defs else mergeEqualOption loc defs;`<sup>2</sup> – deshalb steht unten `EnvironmentFile = [ "${./runner.env}" ];`, nicht `EnvironmentFile = "${./runner.env}";`. systemd selbst unterstützt mehrfache `EnvironmentFile=`-Zeilen ohnehin nativ und liest alle.

## Dateien

`<repo-root>/modules/runner/runner.env` (neu):

```bash
# <repo-root>/modules/runner/runner.env
RUNNER_ENVIRONMENT=produktion
RUNNER_SITE=rz-intern
TZ=Europe/Berlin
```

Bewusst nur Betriebsmetadaten, keine Zugangsdaten. `#`-Zeilen ignoriert systemds `EnvironmentFile`-Parser laut `systemd.exec(5)` ohnehin als Kommentar – die Pfadangabe in Zeile 1 ist also nicht nur Buch-Konvention, sondern in der echten Datei ein gültiger, wirkungsloser Kommentar.<sup>3</sup>

`<repo-root>/modules/runner/forgejo-runner.nix` (geändert):

```diff
--- a/modules/runner/forgejo-runner.nix
+++ b/modules/runner/forgejo-runner.nix
@@
-{ pkgs, ... }:
+{ pkgs, utils, ... }:
 {
   services.gitea-actions-runner = {
@@
       labels = [
         "ubuntu-latest:docker://node:20-bookworm"
       ];
     };
   };
+
+  # utils.escapeSystemdPath berechnet denselben escapten Namen, den das
+  # Runner-Modul intern für die Unit verwendet (Schritt 11) – von Hand
+  # zusammengebaut wäre er bei jedem Bindestrich in <runner-name> falsch.
+  # Als Liste geschrieben, führt der Merge für `unitOption` diesen Eintrag
+  # mit dem bereits vorhandenen EnvironmentFile aus instance.tokenFile
+  # zusammen, statt einen Konflikt zu melden.
+  systemd.services."gitea-runner-${utils.escapeSystemdPath "<runner-name>"}".serviceConfig.EnvironmentFile = [
+    "${./runner.env}"
+  ];
 }
```

`utils` ist wie `pkgs` und `lib` ein Standard-Modulargument, das jedes NixOS-Modul optional entgegennehmen kann – daher der geänderte Funktionskopf. `${./runner.env}` kopiert die Datei beim Bauen automatisch in den Nix Store (Pfad-Typ, Kapitel 3) und interpoliert den resultierenden Store-Pfad als String.

## Prüfen

- `pct exec <vmid> -- systemctl cat 'gitea-runner-*'` zeigt im `[Service]`-Abschnitt **zwei** `EnvironmentFile=`-Zeilen: eine auf `/etc/gitea-runner-<runner-name>-token.env` (Schritt 11), eine auf einen `/nix/store/…-runner.env`-Pfad (dieser Schritt).
- `pct exec <vmid> -- stat -c '%a %U:%G' /nix/store/…-runner.env` (Pfad aus der vorigen Ausgabe) zeigt Modus `444`, Eigentümer `root:root` – lesbar für alle, schreibbar für niemanden. Genau das ist der Grund, warum hier ausschließlich unkritische Werte stehen dürfen.
- Der Dienst bleibt weiterhin `failed`/`activating (auto-restart)` – nicht wegen `runner.env`, sondern weil `/etc/gitea-runner-<runner-name>-token.env` aus Schritt 11 nach wie vor fehlt.

## Wenn's schiefgeht

**`error: undefined variable 'utils'`:** Der Funktionskopf wurde nicht auf `{ pkgs, utils, ... }:` erweitert – `...` allein bindet keine benannten Variablen im Funktionskörper. Fix: Kopfzeile wie im Diff korrigieren.

**Build bricht ab mit *"The option `systemd.services.gitea-runner-…serviceConfig.EnvironmentFile' has conflicting definition values: …"*** (exakter Wortlaut aus `lib/options.nix`, `mergeEqualOption`)<sup>4</sup>: `EnvironmentFile` wurde als einzelner String statt als Liste geschrieben – dann greift nicht der listen-mergende Zweig von `unitOption`, sondern der Gleichheits-Check, und die beiden unterschiedlichen Pfade (Token-Datei, `runner.env`) gelten als Widerspruch. Fix: eckige Klammern nicht vergessen.

**`systemctl cat` zeigt nur eine `EnvironmentFile`-Zeile:** Meist der unescapte Unit-Name in der `systemd.services`-Zuweisung (wörtliche Bindestriche statt `escapeSystemdPath`-Ergebnis) – dann landet die Ergänzung auf einer eigenen, nie verwendeten Unit. Mit `systemctl list-units 'gitea-runner-*' --all` gegenprüfen, dass wirklich nur eine Unit existiert.

## Rückweg

Beide Diff-Hunks aus `forgejo-runner.nix` zurücknehmen (Funktionskopf auf `{ pkgs, ... }:`, `systemd.services."gitea-runner-…"`-Block entfernen) und rebuilden. Der Store-Pfad von `runner.env` bleibt bis zur nächsten `nix-collect-garbage` liegen, wird danach automatisch entfernt – er war nie Teil einer separaten, von Hand zu pflegenden Datei außerhalb des Repos.

## Querverweis

Pfad-Typ und automatische Store-Kopie: Teil I, Kapitel 3 ("Nix als Sprache"). Nix-Store-Berechtigungen (weltlesbar, unveränderlich): Kapitel 2 ("Das Nix-Modell"). Echtes Secrets-Management (sops-age) folgt in Schritt 15 dieses Projekts und ist in Teil I bereits in Kapitel 10 ("Secrets") grundgelegt – dorthin gehört alles, was in dieser `.env` ausdrücklich **nicht** stehen darf.

---

<sup>1</sup> Quelle: Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (Branch `release-26.05`) – Options-Deklaration von `instances.<name>` (`token`, `tokenFile`, `settings`, keine generische Env-Option). https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/continuous-integration/gitea-actions-runner.nix

<sup>2</sup> Quelle: Nixpkgs-Quellcode, `nixos/lib/systemd-unit-options.nix` (Branch `release-26.05`), Definition von `unitOption` (`mkOptionType` mit listen-sensitivem `merge`). https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/lib/systemd-unit-options.nix

<sup>3</sup> Quelle: systemd-Quellcode, `man/systemd.exec.xml` (Branch `main`), Abschnitt `EnvironmentFile=`: "Empty lines, lines without an `=` separator, or lines starting with `;` or `#` will be ignored" sowie "If the file does not exist, cannot be read, or contains invalid content, the service will fail to start. To make the file optional, prefix the path with `-`". https://github.com/systemd/systemd/blob/main/man/systemd.exec.xml

<sup>4</sup> Quelle: Nixpkgs-Quellcode, `lib/options.nix` (Branch `release-26.05`), Funktion `mergeEqualOption`. https://github.com/NixOS/nixpkgs/blob/release-26.05/lib/options.nix
