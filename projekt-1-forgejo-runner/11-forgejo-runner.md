---
title: "Forgejo-Runner-Dienst"
weight: 11
---

# Schritt 11: `services.gitea-actions-runner` läuft mit `pkgs.forgejo-runner`, registriert sich aber noch nicht

## Ziel

`services.gitea-actions-runner` ist vollständig strukturiert – Paket, Instanzname, URL, Labels –, bewusst aber ohne Token. Der erzeugte systemd-Dienst existiert und ist aktiviert, scheitert aber noch am Start, bis Schritt 13 das Token liefert.

## Voraussetzung

Schritt 10 ist abgeschlossen: `virtualisation.podman.enable = true;` steht in `modules/runner/container-runtime.nix`, `modules/runner/default.nix` importiert diese Datei bereits.

## Durchführung

**1. `forgejo-runner.nix` schreiben** (siehe "Dateien") und **2. `default.nix` um eine Zeile erweitern.**

**3. Testen, dann aktivieren:**

```console
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

**Zur Modul-Struktur, bevor der Code Sinn ergibt:** `package` ist im Nixpkgs-Quellcode eine **globale** Option (`options.services.gitea-actions-runner.package = mkPackageOption pkgs "gitea-actions-runner" { };`), keine pro Instanz – ihr Default wäre `pkgs.gitea-actions-runner`, das hier auf `pkgs.forgejo-runner` überschrieben wird, um die Forgejo-eigene Runner-Binary statt der Gitea-Variante zu verwenden.<sup>1</sup> Jede konkrete Registrierung liegt dagegen unter `services.gitea-actions-runner.instances.<name>` (ein `attrsOf submodule`) mit den Feldern `enable`, `name` (Anzeigename bei Forgejo), `url`, `token`/`tokenFile` (genau eines von beiden, siehe unten), `labels`, `settings` (freiform YAML für den Runner-Daemon, hier ungenutzt) und `hostPackages` (nur relevant für `:host`-Labels, Default bereits `bash`/`coreutils`/`curl`/`gawk`/`gitMinimal`/`gnused`/`nodejs`/`wget`).<sup>1</sup> Der Instanzname ist laut Dateikontrakt `<runner-name>`.

**Warum `tokenFile` und nicht `token`:** Eine Assertion im Modul verlangt genau eines von beiden – wörtlich: *"Instances of gitea-actions-runner can have `token` or `tokenFile`, not both."*<sup>1</sup> `tokenFile` verweist unten auf `/etc/gitea-runner-<runner-name>-token.env`, eine Datei, die **noch nicht existiert** – das ist bewusst so, siehe "Prüfen". Schritt 13 legt sie mit dem Inhalt `TOKEN=<registrierungstoken>` an; erst dann kann der Dienst tatsächlich starten. Ohne dieses Feld bräche schon der Build ab, weil die Assertion fehlschlägt – mit einer gesetzten, aber (noch) nicht existenten Datei baut alles, nur der *Dienststart* schlägt fehl (siehe Prüfen).

**Zur Label-Syntax:** `<label>:docker://<image>` weist einer Job-Umgebung (dem `runs-on:`-Label im Workflow) ein Container-Image zu, das Podman/Docker vor jedem Job zieht; `<label>:host` führt den Job stattdessen direkt auf diesem Host aus, ohne Container – dafür sind dann `hostPackages` im `$PATH`. Eine leere `labels`-Liste ist laut Kommentar im Quellcode gleichbedeutend mit den vom Upstream-Runner selbst vorgegebenen Default-Labels, die zwingend Docker/Podman voraussetzen.<sup>1</sup> Genau deshalb verlangt eine zweite Assertion einen aktiven Container-Runtime, sobald irgendein Label `:docker:` enthält (oder die Liste leer ist) – wörtlich: *"Label configuration on gitea-actions-runner instance requires either docker or podman."*<sup>1</sup> Schritt 10 erfüllt das bereits.

## Dateien

`<repo-root>/modules/runner/forgejo-runner.nix` (neu):

```nix
# <repo-root>/modules/runner/forgejo-runner.nix
{ pkgs, ... }:
{
  services.gitea-actions-runner = {
    package = pkgs.forgejo-runner;

    instances."<runner-name>" = {
      enable = true;
      name = "<runner-name>";
      url = "<forgejo-url>";

      # Existiert noch nicht – Schritt 13 legt diese Datei mit
      # `TOKEN=<registrierungstoken>` an. Bis dahin startet der Dienst
      # bewusst nicht (siehe Schritt-Text, Abschnitt "Prüfen").
      tokenFile = "/etc/gitea-runner-<runner-name>-token.env";

      labels = [
        "ubuntu-latest:docker://node:20-bookworm"
      ];
    };
  };
}
```

`<repo-root>/modules/runner/default.nix` (geändert, eine Zeile):

```diff
--- a/modules/runner/default.nix
+++ b/modules/runner/default.nix
@@
   imports = [
     ./container-runtime.nix
+    ./forgejo-runner.nix
   ];
```

> 💡 **Nice to know:** Der erzeugte systemd-Unit-Name ist nicht einfach `gitea-runner-<runner-name>.service`, sondern `gitea-runner-${escapeSystemdPath name}.service` – `escapeSystemdPath` behandelt den Instanznamen wie einen Pfad und ersetzt dabei u. a. jeden literalen Bindestrich durch `\x2d` (Bindestrich ist im Escaping-Schema selbst reserviert, weil er nach der Slash-zu-Bindestrich-Umwandlung als Trenner gilt).<sup>2</sup> Für den Beispielwert `ci-runner-01` aus der Platzhaltertabelle lautet die reale Unit also `gitea-runner-ci\x2drunner\x2d01.service` – ein reiner Buchstaben-/Ziffern-Name ohne Bindestrich bliebe dagegen unverändert. Am robustesten: nicht von Hand zusammenbauen, sondern mit einem Glob suchen (siehe "Prüfen").

## Prüfen

- `nixos-rebuild build`/`switch` laufen ohne Assertion-Fehler durch.
- `pct exec <vmid> -- systemctl status 'gitea-runner-*'` zeigt genau eine Unit mit dem oben beschriebenen (escapten) Namen – Zustand **`failed`** oder oszillierend zwischen `activating (auto-restart)` und `failed` (`Restart = "on-failure"`, `RestartSec = 2` stehen im Modul selbst). Das ist für diesen Schritt der *korrekte* Zustand, kein Fehler.
- `pct exec <vmid> -- journalctl -u 'gitea-runner-*' -n 20` zeigt, dass der Start an der fehlenden `/etc/gitea-runner-<runner-name>-token.env` scheitert (`EnvironmentFile=`-Verarbeitung, bevor der eigentliche Registrierungs-Code überhaupt läuft).
- `pct exec <vmid> -- ls -l /etc/gitea-runner-<runner-name>-token.env` bestätigt: Datei existiert (noch) nicht.

## Wenn's schiefgeht

**`error: The option `services.gitea-actions-runner.instances."<runner-name>".labels' was accessed but has no value defined. Try setting the option.`** (exakter Wortlaut aus `lib/modules.nix`)<sup>3</sup>: `labels` hat im Modul keinen Default – anders als `token`/`tokenFile` ist es Pflicht. Fix: mindestens ein Label setzen, notfalls `labels = [ ];` (bedeutet dann implizit die Upstream-Default-Labels, siehe oben).

**Build bricht mit *"Instances of gitea-actions-runner can have `token` or `tokenFile`, not both."* ab:** Entweder beide Felder gesetzt (z. B. versehentlich `token = "";` stehen gelassen) oder beide `null`. Fix: genau `tokenFile` wie oben setzen, `token` weglassen.

**`systemctl status gitea-runner-<runner-name>` (unescaped, Bindestriche wörtlich) meldet, die Unit sei nicht auffindbar:** Erwartetes Verhalten, siehe Nice-to-know-Box oben – der reale Name ist escaped. Fix: `systemctl status 'gitea-runner-*'` oder `systemd-escape --path <runner-name>` zur Kontrolle.

## Rückweg

`./forgejo-runner.nix` aus `modules/runner/default.nix` entfernen und rebuilden – ohne `instances` greift `config = mkIf (cfg.instances != { }) { … };` im Modul gar nicht mehr, der Dienst verschwindet vollständig (keine Unit, keine Assertions).

## Querverweis

Attribute-Sets von Submodulen (`instances.<name>`) und das `mkIf`-Muster für optionale Modul-Teile: Teil I, Kapitel 5 ("Das Modulsystem"). `systemd.services`, `Restart`/`RestartSec`: Kapitel 6 ("Alltagsbetrieb"). CI-Runner selbst kommt in Teil I nicht vor – reines Projekt-II-Terrain.

---

<sup>1</sup> Quelle: Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (Branch `release-26.05`), verbatim geladen über `raw.githubusercontent.com` – Options-Deklaration (`package`, `instances.<name>.*`), beide `assertions`-Einträge im Wortlaut, Kommentar zu leeren `labels` ("Empty label strings result in the upstream defined defaultLabels, which require docker"). https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/continuous-integration/gitea-actions-runner.nix

<sup>2</sup> Quelle: Nixpkgs-Quellcode, `nixos/lib/utils.nix` (Branch `release-26.05`), Funktion `escapeSystemdPath`: Zeichenklasse `" !\"#$%&'()*+,;<=>?@[\\]^\`{|}~-"` wird über `escapeC` in `\xNN`-Sequenzen umgewandelt (`-` ist Teil dieser Klasse), danach werden `/` zu `-`. https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/lib/utils.nix

<sup>3</sup> Quelle: Nixpkgs-Quellcode, `lib/modules.nix` (Branch `release-26.05`), Zeile mit der Fehlermeldung für Optionen ohne `default`, die nie definiert werden: `"The option \`${showOption loc}' was accessed but has no value defined. Try setting the option."`. https://github.com/NixOS/nixpkgs/blob/release-26.05/lib/modules.nix
