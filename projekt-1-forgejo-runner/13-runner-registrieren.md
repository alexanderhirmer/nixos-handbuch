---
title: "Runner registrieren"
weight: 13
---

# Schritt 13: Der Runner ist mit einem Forgejo-Token registriert und online

## Ziel

`services.gitea-actions-runner.instances."<runner-name>"` bekommt über `tokenFile` (nicht `token`) ein in der Forgejo-Weboberfläche erzeugtes Registrierungs-Token. Der Runner registriert sich beim ersten Start selbstständig bei `<forgejo-url>` und erscheint dort als online.

## Voraussetzung

Schritte 10–12 sind abgeschlossen: Podman läuft, `modules/runner/forgejo-runner.nix` definiert die Instanz bereits mit `name`, `url`, `labels`, aber ohne `token`/`tokenFile` – ein `nixos-rebuild build` bricht deshalb an dieser Stelle mit der nixpkgs-eigenen Assertion ab: *"Instances of gitea-actions-runner can have `token` or `tokenFile`, not both."*<sup>1</sup> (dieselbe Meldung erscheint auch, wenn – wie hier – **keins** von beiden gesetzt ist). `modules/runner/runner.env` (Schritt 12) ist unabhängig davon und bleibt unverändert. Netzwerkzugriff auf `<forgejo-url>` besteht bereits (Schritt 9, ausgehend unbeschränkt).

## Durchführung

**1. Geltungsbereich wählen.** Forgejo registriert Runner instanzweit, auf Organisations-, Nutzer- oder Repository-Ebene – je enger, desto weniger Repos darf der Runner bedienen.<sup>2</sup> Da dieser Container laut Übersicht ein Allzweck-CI-Runner für die ganze Instanz sein soll, passt die **instanzweite** Registrierung.

> ⚠️ Ungeprüft: `forgejo.org` war aus dieser Umgebung heraus per Egress-Policy vollständig blockiert, auch lesend. Der folgende Pfad stützt sich auf mehrere unabhängige Sekundärquellen, nicht auf eigene Ansicht der Primärquelle: **Site Administration → Actions → Runners → "Create new runner"** unter `/admin/actions/runners` (Stand Forgejo v16.0 LTS, August 2026). Organisations-Scope: `/org/<org>/settings/actions/runners`, Repository-Scope: `/<owner>/<repo>/settings/actions/runners`. Vor Übernahme selbst gegenprüfen.

**2. Runner anlegen, Token kopieren.** Name/Beschreibung eingeben; Forgejo zeigt UUID und Token **genau einmal** an.

**3. Token-Datei außerhalb des Stores anlegen** – dasselbe Muster wie in Schritt 8 für das LDAP-Bind-Passwort:

```console
$ pct exec <vmid> -- sh -c 'umask 077; printf "TOKEN=%s\n" "<token-aus-weboberflaeche>" > /etc/gitea-runner-token.env'
$ pct exec <vmid> -- chown root:root /etc/gitea-runner-token.env
$ pct exec <vmid> -- chmod 600 /etc/gitea-runner-token.env
```

**4. `forgejo-runner.nix` um `tokenFile` ergänzen**, pushen, testen, aktivieren:

```console
$ pct push <vmid> <repo-root>/modules/runner/forgejo-runner.nix \
    /etc/nixos/modules/runner/forgejo-runner.nix
$ pct enter <vmid>
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

## Dateien

`<repo-root>/modules/runner/forgejo-runner.nix` (geändert, um eine Zeile erweitert):

```diff
--- a/modules/runner/forgejo-runner.nix
+++ b/modules/runner/forgejo-runner.nix
@@
   services.gitea-actions-runner.instances."<runner-name>" = {
     enable = true;
     name = "<runner-name>";
     url = "<forgejo-url>";
+    # String, KEIN Nix-Pfad-Literal (./token.env würde die Datei in den
+    # – world-readable! – Store kopieren, siehe Kapitel 10). Die Datei
+    # muss auf dem Zielsystem bereits existieren, bevor der Dienst startet.
+    tokenFile = "/etc/gitea-runner-token.env";
     labels = [ /* … aus Schritt 11, hier unverändert … */ ];
   };
```

`token` wäre die Alternative – landet damit aber wortwörtlich in der generierten Unit-Datei unter `/nix/store/…-gitea-runner-<runner-name>.service`, für jeden lokalen Nutzer lesbar.<sup>3</sup> `tokenFile` verweist stattdessen auf `EnvironmentFile=`; diese Datei liest der systemd-**Manager** (PID 1, root) unmittelbar vor `fork`/`exec`, **bevor** er auf den in `User = "gitea-runner"` (`DynamicUser = true`) angegebenen unprivilegierten Nutzer wechselt.<sup>4</sup> `chmod 600 root:root` genügt deshalb.

> 💡 **Nice to know:** Die systemd-Unit heißt nicht `gitea-runner-<runner-name>.service`. Nixpkgs baut den Namen über `escapeSystemdPath`,<sup>5</sup> und dieses Escaping maskiert einen literalen Bindestrich zu `\x2d` (in systemd-Unit-Namen für Pfadtrennung reserviert). Aus `ci-runner-01` wird real `gitea-runner-ci\x2drunner\x2d01.service` – nachvollziehbar mit `systemd-escape ci-runner-01`. Vor `systemctl status`/`journalctl -u` lohnt sich `systemd-escape "<runner-name>"`, statt den Namen zu raten.

## Prüfen

- In der Forgejo-Oberfläche (derselbe Pfad wie bei der Registrierung) erscheint `<runner-name>` mit Online-Status (⚠️ Ungeprüft: exakter UI-Text).
- `pct exec <vmid> -- systemctl status "gitea-runner-$(systemd-escape '<runner-name>')"` zeigt `active (running)`.
- `pct exec <vmid> -- journalctl -u "gitea-runner-$(systemd-escape '<runner-name>')" -e` zeigt einen erfolgreichen Registrierungslauf, danach dauerhaftes Laufen ohne Neustart-Schleife.
- `pct exec <vmid> -- test -e "/var/lib/gitea-runner/<runner-name>/.runner"` liefert Exit-Code 0.

## Wenn's schiefgeht

**Build bricht weiterhin mit der "not both"-Assertion ab:** `token` ist zusätzlich irgendwo gesetzt, oder `tokenFile` wird in einer anderen Moduldatei überschrieben. `modules/runner/*.nix` auf doppelte Zuweisungen prüfen.

**Unit startet nicht, `journalctl` meldet ein Problem beim Laden der Environment-Datei:** `/etc/gitea-runner-token.env` fehlt oder Pfad ist falsch – `EnvironmentFile=` erzwingt einen Fehlschlag, wenn die Datei fehlt oder unlesbar ist.<sup>4</sup> Mit `pct exec <vmid> -- ls -l /etc/gitea-runner-token.env` gegenprüfen.

**Unit startet, `ExecStartPre` (Registrierung) schlägt fehl, der Dienst rotiert ständig:** Meist ein ungültiges oder verbrauchtes Token. Neues Token erzeugen, `/etc/gitea-runner-token.env` ersetzen – der veränderte Inhalt zwingt beim nächsten Start automatisch zur Neuregistrierung.<sup>1</sup> Ein reiner `systemctl restart` **ohne** Token-Änderung registriert dagegen nicht neu, solange `.runner` existiert und Labels/Token unverändert sind – Absicht: Die Laufzeit-Authentifizierung nutzt danach ein bei der Registrierung ausgehandeltes, vom Registrierungs-Token verschiedenes Credential in `.runner`.

## Rückweg

`tokenFile`-Zeile aus `forgejo-runner.nix` entfernen, pushen, rebuilden – der Dienst stoppt. Das ändert nichts auf Forgejo-Seite: Der Eintrag bleibt sichtbar (als offline), bis er dort manuell gelöscht wird. Für einen sauberen Neuanfang zusätzlich `pct exec <vmid> -- rm -rf /var/lib/gitea-runner/<runner-name>` und `/etc/gitea-runner-token.env` löschen.

## Querverweis

Store-Welt-Lesbarkeit: Teil I, Kapitel 10 ("Secrets"). Das Muster "Zugangsdaten außerhalb des Repos, `chmod 600`" stammt aus Schritt 8 (LDAP-Bind). Schritt 15 ersetzt eine vergleichbare Klartextdatei durch einen sops-age-verschlüsselten Mechanismus.

---

<sup>1</sup> Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (Branch `release-26.05`): Assertion-Text, `tokenXorTokenFile`, `EnvironmentFile = instance.tokenFile`, `ExecStartPre`-Registrierungsskript mit `.runner`/`.token-hash`/`.labels`-Markerdateien unter `$STATE_DIRECTORY/${name}` (= `/var/lib/gitea-runner/<name>`). https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/continuous-integration/gitea-actions-runner.nix

<sup>2</sup> Sekundärquellen (Zitat/Zusammenfassung der offiziellen, hier nicht direkt erreichbaren Forgejo-Doku "Runner Registration"): instanzweit unter `/admin/actions/runners` (alle Repos der Instanz), Organisation `/org/{org}/settings/actions/runners`, Nutzer `/user/settings/actions/runners`, Repository `/{owner}/{repo}/settings/actions/runners`. Dasselbe Token registriert laut derselben Quelle mehrere unabhängige Runner-Prozesse – kein Single-Use-Token.

<sup>3</sup> Store-Pfade sind für jeden lokalen Nutzer lesbar (Teil I, Kapitel 2 und 10). NixOS-Systemd-Units werden als Store-Ableitungen gebaut und über `/etc/systemd/system/` verlinkt.

<sup>4</sup> `systemd.exec`(5) (Abschnitt `EnvironmentFile=`) sowie systemd-Quellcode `src/core/execute.c`, Funktion `exec_spawn()`: `exec_context_load_environment()` läuft dort, **bevor** der Kindprozess erzeugt und `User=`/`DynamicUser=` angewendet werden – die Datei wird mit den Rechten des Managers (root) gelesen, nicht mit denen des späteren Dienst-Nutzers. https://github.com/systemd/systemd/blob/main/src/core/execute.c

<sup>5</sup> Nixpkgs-Quellcode, `nixos/lib/utils.nix` (Branch `release-26.05`), Funktion `escapeSystemdPath` – Bindestrich ist Teil des zu escapenden Zeichensatzes; gegenprüft mit dem realen `systemd-escape`-Kommando (`systemd-escape ci-runner-01` → `ci\x2drunner\x2d01`). https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/lib/utils.nix
