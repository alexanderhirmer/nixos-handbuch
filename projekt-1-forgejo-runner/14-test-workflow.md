---
title: "Test-Workflow"
weight: 14
---

# Schritt 14: Ein Test-Workflow läuft erfolgreich über den neuen Runner

## Ziel

Ein minimaler Workflow in einem **Test-Repository auf der Forgejo-Instanz** wird durch einen Push ausgelöst, vom neu registrierten Runner (Schritt 13) angenommen und läuft mit grünem Status durch.

## Voraussetzung

Schritt 13 ist abgeschlossen: `<runner-name>` erscheint online. Ein Test-Repository existiert bereits auf `<forgejo-url>` (Git-Hosting selbst ist laut Projektrahmen bestehende Infrastruktur, nicht Teil dieses Projekts). **Wichtig:** Dieses Test-Repository ist ein völlig anderer Ort als `<repo-root>` – das NixOS-Infrastruktur-Repo aus den Schritten 1–13. Die hier angelegte Workflow-Datei gehört **ausschließlich** ins Test-Repository und hat mit `<repo-root>` nichts zu tun.

## Durchführung

**1. Verzeichnis wählen.** Forgejo sucht zuerst unter `.forgejo/workflows/`; existiert dieses Verzeichnis nicht, fällt es auf `.github/workflows/` zurück – sind **beide** vorhanden, führt Forgejo Workflows aus beiden aus (anders als GitHub, das `.forgejo/workflows` ignoriert).<sup>1</sup> Für ein Forgejo-natives Repo gehört der Workflow deshalb nach `.forgejo/workflows/`.

**2. Labels gegenprüfen.** `runs-on:` muss exakt den Namensteil **vor** dem Doppelpunkt eines der in `forgejo-runner.nix` (Schritt 11) registrierten Labels treffen (Groß-/Kleinschreibung zählt) – z. B. macht ein Label `debian-trixie:docker://…` den Wert `runs-on: debian-trixie` gültig, `debian-latest` dagegen nicht. Die registrierten Labels stehen sowohl im Nix-Modul als auch (nach Schritt 13) in der Forgejo-Runner-Übersicht.

**3. Bei einem `:docker:`-Schema-Label das Image lokal verfügbar machen.** Dieses Projekt läuft in einem rein internen Netz ohne öffentlichen Zugriff (Projektrahmen, `00-uebersicht.md`). Ein Image direkt von einer öffentlichen Registry zu ziehen, scheitert dort ohne Weiteres. Sofern keine interne Registry/Spiegelung bereitsteht, das Image vorab manuell cachen:

```console
$ pct exec <vmid> -- podman pull docker.io/library/debian:trixie-slim
```

**4. Workflow-Datei anlegen** (siehe "Dateien"), committen, ins Test-Repository pushen.

**5. Lauf beobachten:** im Test-Repository unter "Actions" erscheint der ausgelöste Lauf, zugeordnet zu `<runner-name>`.

## Dateien

`.forgejo/workflows/runner-test.yaml` (neu, **im Test-Repository**, nicht in `<repo-root>`):

```yaml
# .forgejo/workflows/runner-test.yaml
name: runner-test
on:
  push:
  workflow_dispatch:

jobs:
  hello:
    runs-on: debian-trixie
    steps:
      - uses: actions/checkout@v4
      - run: |
          echo "Runner-Hostname im Job: $(hostname)"
          echo "Repo-Inhalt nach Checkout:"
          ls -la
```

`actions/checkout@v4` wird nicht direkt von GitHub geladen: Fehlt eine vollständige URL, hängt Forgejo den in `DEFAULT_ACTIONS_URL` konfigurierten Präfix davor (Default laut Doku `https://data.forgejo.org`, vom Instanz-Admin änderbar, z. B. auf `https://github.com` oder eine selbst gespiegelte Quelle).<sup>2</sup> Ob das innerhalb des internen Netzes dieses Projekts erreichbar ist, hängt von der Konfiguration der – laut Projektrahmen bereits bestehenden – Forgejo-Instanz ab, nicht von diesem Runner.

> 💡 **Nice to know – die häufigste Verwirrung:** Bei einem `:docker:`-Schema-Label läuft der Job **innerhalb des angegebenen Container-Images**, nicht in der NixOS-Umgebung des Runner-Hosts. `$(hostname)`, installierte Pakete, sogar die Linux-Distribution im Job sind die des Images (hier Debian, nicht NixOS) – nichts aus `environment.systemPackages` des Hosts ist automatisch verfügbar. Nur bei einem `:host`-Schema-Label (z. B. `irgendwas:host`) läuft der Job direkt im Dateisystem des Runner-Containers; dann – und nur dann – zählt `services.gitea-actions-runner.instances.<runner-name>.hostPackages`, das laut Nixpkgs-Quellcode standardmäßig `bash`, `coreutils`, `curl`, `gawk`, `gitMinimal`, `gnused`, `nodejs` und `wget` auf den `PATH` des Jobs legt – alles andere muss dort explizit ergänzt werden.<sup>3</sup>

## Prüfen

- Im Test-Repository unter "Actions" zeigt der Lauf von `runner-test` einen grünen/erfolgreichen Status.
- Das Job-Log enthält die Zeile `Runner-Hostname im Job: …` sowie eine Verzeichnisliste, die tatsächlich den Inhalt des Test-Repos zeigt – Beleg, dass `actions/checkout` erfolgreich aufgelöst und ausgeführt wurde.
- `pct exec <vmid> -- journalctl -u "gitea-runner-$(systemd-escape '<runner-name>')" --since -10m` zeigt Aktivität während des Laufs.

## Wenn's schiefgeht

**Der Lauf bleibt dauerhaft auf "Waiting"/wartet auf einen Runner:** Label-Mismatch – kein registrierter Runner bietet das in `runs-on:` verlangte Label an. `forgejo-runner.nix` (Schritt 11) und den Wert in `runs-on:` exakt vergleichen, auch auf Groß-/Kleinschreibung.

**Der Job scheitert im Schritt "Set up job", die Action lässt sich nicht auflösen:** Typisches, in Forgejo-Issues dokumentiertes Fehlerbild ist ein Timeout/Fehlercode beim Holen der Action-Quelle, z. B. sinngemäß *"unexpected requesting '…/actions/checkout/info/refs?service=git-upload-pack' status code: 502"*<sup>4</sup> – ein Zeichen, dass die konfigurierte `DEFAULT_ACTIONS_URL` vom Container aus nicht erreichbar ist (naheliegend im rein internen Netz dieses Projekts). Workaround unabhängig von der Instanz-Konfiguration: die Action mit voll qualifizierter URL referenzieren, z. B. `uses: https://github.com/actions/checkout@v4` (nur nutzbar, wenn dieser Host tatsächlich erreichbar ist) oder eine intern erreichbare Spiegelung angeben.

**Container-Start scheitert, Log zeigt einen Pull-Fehler:** Image nicht lokal vorhanden und Registry im internen Netz nicht erreichbar. Mit `podman pull` wie oben vorab cachen, oder auf ein `:host`-Schema-Label ausweichen, das keinen Container-Pull braucht.

## Rückweg

Workflow-Datei im Test-Repository löschen oder den Branch/Commit verwerfen – ohne Auswirkung auf `<repo-root>` oder die NixOS-Konfiguration des Runners selbst.

## Querverweis

Podman als Container-Runtime: Schritt 10. Labels und Registrierung: Schritt 11 und 13. "Rein internes Netz, kein öffentlicher Zugriff": Projektrahmen, `00-uebersicht.md`. Firewall/ausgehende Verbindungen: Schritt 9.

---

<sup>1</sup> Sekundärquellen (Zusammenfassung der in dieser Umgebung nicht direkt erreichbaren Forgejo-Dokumentation zu Actions): `.forgejo/workflows/` hat Vorrang; existiert es nicht, wird `.github/workflows/` verwendet; sind beide vorhanden, führt Forgejo (anders als GitHub) Workflows aus beiden Verzeichnissen aus.

<sup>2</sup> Sekundärquellen (Zusammenfassung der offiziellen, hier nicht direkt erreichbaren Forgejo-Doku zu `DEFAULT_ACTIONS_URL`): Default `https://data.forgejo.org`, änderbar in `app.ini` unter `[actions]`, u. a. auf `https://github.com` oder `self` (lokale Spiegelung mit identischer Organisations-/Repo-Struktur).

<sup>3</sup> Nixpkgs-Quellcode, `nixos/modules/services/continuous-integration/gitea-actions-runner.nix` (Branch `release-26.05`), Option `hostPackages` (Default-Liste) sowie `path = with pkgs; [ coreutils ] ++ lib.optionals wantsHost instance.hostPackages;` im generierten systemd-Unit. https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/continuous-integration/gitea-actions-runner.nix

<sup>4</sup> Sekundärquelle: Diskussion in einem Forgejo-Issue zu einem `DEFAULT_ACTIONS_URL`-Bug (Codeberg, `forgejo/forgejo`), in dieser Umgebung nicht direkt einsehbar – Fehlertext hier als Zitat einer Sekundärquelle übernommen, nicht selbst am Gerät reproduziert. ⚠️ Ungeprüft im Sinne dieses Buchs: exakter Wortlaut und ob er auf v16.0 noch zutrifft.
