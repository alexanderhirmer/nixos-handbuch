---
title: "Passwortloses Sudo"
weight: 4
---

# Schritt 4: Nur `<admin-user>` sudoet ohne Passwort

## Ziel

`security.sudo.extraRules` gewährt ausschließlich `<admin-user>` passwortloses `sudo`; alle anderen künftigen `wheel`-Mitglieder (die LDAP-Admins aus Schritt 8) sudoen weiterhin mit Passwort.

## Voraussetzung

Schritt 3 ist abgeschlossen: `<admin-user>` existiert, ist Mitglied von `wheel` und kann sich per SSH-Key anmelden (verifiziert bislang nur über `pct exec`, da Port/Firewall noch offen sind).

## Durchführung

**1. `sudo.nix` schreiben** (siehe "Dateien") und in `modules/baseline/default.nix` verdrahten.

**2. Pushen und aktivieren:**

```console
$ pct push <vmid> <repo-root>/modules/baseline/sudo.nix \
    /etc/nixos/modules/baseline/sudo.nix
$ pct push <vmid> <repo-root>/modules/baseline/default.nix \
    /etc/nixos/modules/baseline/default.nix
$ pct enter <vmid>
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

## Dateien

```nix
# <repo-root>/modules/baseline/sudo.nix
{ ... }:
{
  # security.sudo.wheelNeedsPassword bleibt bewusst beim NixOS-Default
  # `true` – sonst würden alle wheel-Mitglieder passwortlos sudoen,
  # nicht nur <admin-user>. Diese Regel gilt gezielt nur für einen Nutzer.
  security.sudo.extraRules = [
    {
      users = [ "<admin-user>" ];
      commands = [
        {
          command = "ALL";
          options = [ "NOPASSWD" ];
        }
      ];
    }
  ];
}
```

```diff
--- a/modules/baseline/default.nix
+++ b/modules/baseline/default.nix
@@
   imports = [
     ./users.nix
+    ./sudo.nix
   ];
```

**Zur Reihenfolge, nicht nur zur Regel:** `security.sudo.extraRules` ist listenwertig – NixOS' eigenes `security.sudo`-Modul trägt dort selbst schon zwei Einträge ein: eine Regel für `root` (mit `lib.mkOrder 400`) und eine für die Gruppe `wheel` (mit `lib.mkOrder 600`, `NOPASSWD` nur falls `wheelNeedsPassword = false` – Default ist `true`, also Passwortpflicht).<sup>1</sup> Eigene, ohne `mkBefore`/`mkOrder` geschriebene Einträge bekommen automatisch die Standard-Ordnungszahl 1000 und landen damit in der zusammengeführten Liste – und später in `/etc/sudoers` – *nach* der `wheel`-Regel. `sudo` selbst wertet `/etc/sudoers` von oben nach unten aus; bei mehreren passenden Zeilen für denselben Nutzer gewinnt die letzte.<sup>1</sup> Weil `<admin-user>` sowohl unter die `%wheel`-Zeile (Passwort nötig) als auch unter die eigene, weiter unten stehende Zeile (`NOPASSWD`) fällt, gewinnt Letztere – ganz ohne `security.sudo.wheelNeedsPassword` global anzufassen. Die Optionsbeschreibung von `extraRules` formuliert das Prinzip selbst so: "More specific rules should come after more general ones in order to yield the expected behavior."

> 💡 **Nice to know:** Genau dieses Ordnungsprinzip – reihenfolgeabhängiges Zusammenführen listenwertiger Optionen – ist dasselbe, das Kapitel 5 unter `mkBefore` einführt. `security.sudo`s Modul nutzt intern `mkOrder` (die allgemeinere Form dahinter), nicht `mkBefore` selbst, aber das Konzept ist identisch: die Ordnungszahl entscheidet über die Position in der zusammengeführten Liste, nicht über "wer gewinnt bei Konflikt" im `mkForce`-Sinn.

## Prüfen

`pct exec <vmid> -- sudo -l -U <admin-user>` muss unter den erlaubten Befehlen eine Zeile mit `NOPASSWD: ALL` zeigen. `pct exec <vmid> -- cat /etc/sudoers` zeigt drei relevante Zeilen in dieser Reihenfolge: zuerst `root`, dann `%wheel` (ohne `NOPASSWD`, weil `wheelNeedsPassword` auf `true` steht), zuletzt die Zeile für `<admin-user>` mit `NOPASSWD`. Sobald ab Schritt 8 ein LDAP-Testnutzer in `wheel` ist, muss dessen `sudo -l` weiterhin nach einem Passwort fragen.

## Wenn's schiefgeht

**`sudo -l -U <admin-user>` zeigt keine `NOPASSWD`-Zeile:** Entweder ein Tippfehler im Nutzernamen in `sudo.nix`, oder die Datei fehlt im `imports` von `default.nix` (die Regel existiert dann schlicht nicht im gebauten System). Fix: Schreibweise gegen `users.users."<admin-user>"` aus Schritt 3 abgleichen, `imports`-Zeile prüfen.

**`sudo` fragt trotz vorhandener Regel nach einem Passwort:** Meist eine aus einem Tutorial übernommene `lib.mkBefore`-Verpackung um die eigene Regel. Das schiebt sie vor die eingebaute `%wheel`-Regel (Order 600 statt der eigenen 1000) statt danach – die spätere, passwortpflichtige `%wheel`-Zeile gewinnt dann als letzte passende Regel. Fix: `mkBefore`/`mkOrder` entfernen, die Standard-Reihenfolge reicht.

**Versehentlich `security.sudo.wheelNeedsPassword = false;` gesetzt (z. B. beim Kopieren aus einer Anleitung):** Kein Build-Fehler, aber ein Verstoß gegen das Sicherheitsmodell – plötzlich sudoen alle `wheel`-Mitglieder passwortlos, nicht nur `<admin-user>`. Fix: Zeile entfernen bzw. sicherstellen, dass sie nirgends im Modulbaum gesetzt wird.

## Rückweg

Entweder den `security.sudo.extraRules`-Block aus `sudo.nix` leeren (`[ ]`) oder die `./sudo.nix`-Zeile aus `modules/baseline/default.nix` entfernen, pushen, rebuilden. `environment.etc.sudoers` wird bei jedem `switch` komplett neu aus der aktuellen Konfiguration erzeugt – die `NOPASSWD`-Zeile für `<admin-user>` verschwindet dann vollständig, `sudo` verlangt für diesen Nutzer wieder ein (nicht gesetztes) Passwort, sofern er weiterhin in `wheel` ist.

## Querverweis

Listenwertige Optionen, `mkBefore` und Zusammenführungs-Reihenfolge: Teil I, Kapitel 5 ("Das Modulsystem"). `wheel`-Gruppe und Sudo-Grundprinzip: Kapitel 6 ("Alltagsbetrieb").

---

<sup>1</sup> Quelle: Nixpkgs-Quellcode, `nixos/modules/security/sudo.nix` (Branch `release-26.05`), verbatim geladen über `raw.githubusercontent.com` – Default-Regeln für `root` (`mkOrder 400`) und `wheel` (`mkOrder 600`, `wheelNeedsPassword`-Option), `extraRules`-Beschreibungstext ("More specific rules should come after more general ones …"), Rendering nach `environment.etc.sudoers` via `visudo -c`. https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/security/sudo.nix
