---
title: "Mehrere Maschinen"
weight: 14
---

# Mehrere Maschinen

## Lernziele

- Du kannst `nixos-rebuild --target-host` für einfache Remote-Deploys ohne Zusatztools einsetzen.
- Du kannst `nixos-anywhere` für die bare-metal-Erstprovisionierung über SSH nutzen.
- Du ordnest deploy-rs und seinen Rollback-Mechanismus ein.
- Du ordnest colmena für paralleles Deployment auf mehrere Hosts ein.
- Du wählst anhand einer Vergleichstabelle das passende Tool für eine gegebene Situation.

## Warum das wichtig ist

Alles bisher in diesem Buch war eine einzelne Maschine. Sobald es mehrere werden, reicht "einloggen und `nixos-rebuild switch`" pro Maschine nicht mehr – dafür existieren die vier Werkzeuge dieses Kapitels, mit unterschiedlichen Stärken je nach Situation.

## `nixos-rebuild --target-host`: ohne Zusatztools

Der eingebaute, werkzeuglose Weg, schon aus Kapitel 7 bekannt:

```console
$ nixos-rebuild switch --flake .#my-nixos \
    --target-host root@192.168.4.1 --build-host localhost --verbose
```

`--build-host` legt fest, *wo* gebaut wird (hier lokal, dann übertragen), `--target-host`, *wohin* aktiviert wird. Läuft der Zielnutzer nicht als root, kommt `--use-remote-sudo` dazu. Für passwortlosen, nicht-interaktiven sudo-Zugriff brauchst du auf der Zielmaschine `security.sudo.wheelNeedsPassword = false;` – aus Sicherheitsgründen unbedingt für einen *dedizierten* Deploy-Nutzer, nicht für deinen persönlichen Account, sonst hat jedes Programm, das als dieser Nutzer läuft, stillschweigend Root-Rechte.

Der Vorteil: kein Zusatztool, funktioniert für eine oder zwei Maschinen hervorragend. Der Nachteil: keine eingebaute Parallelisierung – bei zehn Maschinen tippst du den Befehl zehnmal oder scriptest es dir selbst.

## `nixos-anywhere`: bare-metal-Provisionierung über SSH

Aus Kapitel 4 schon angekündigt: `nixos-anywhere` bringt eine Maschine, die nur per SSH erreichbar ist (z. B. über ein generisches Rescue-System eines Hosters), komplett unbeaufsichtigt auf NixOS – kombiniert mit `disko` (Kapitel 4) für die Partitionierung. Das ist explizit ein *Erstprovisionierungs*-Werkzeug, kein Alltags-Update-Tool: Einmal installiert, übernehmen `nixos-rebuild --target-host`, deploy-rs oder colmena das laufende Update-Geschäft.

## deploy-rs

```console
$ nix run github:serokell/deploy-rs .#my-node
```

Ein Flake-natives Deploy-Tool mit zwei Besonderheiten gegenüber dem eingebauten `--target-host`-Weg: Erstens unterstützt es *mehrere Profile pro Node* (z. B. System- und Nutzerprofil getrennt deploybar). Zweitens – der eigentliche Grund, warum es oft gewählt wird – ein eingebauter Sicherheitsmechanismus, umgangssprachlich "Magic Rollback" genannt: Nach der Aktivierung muss sich die Zielmaschine innerhalb eines Zeitfensters selbst als "gesund" zurückmelden; bleibt diese Bestätigung aus (weil z. B. SSH nach der neuen Konfiguration gar nicht mehr erreichbar ist), rollt deploy-rs automatisch zur vorherigen Generation zurück – ganz ohne dass jemand manuell eingreifen muss.

## colmena

```nix
# flake.nix, Ausschnitt
colmena = {
  meta.nixpkgs = import nixpkgs { system = "x86_64-linux"; };

  web1 = {
    deployment = {
      targetHost = "web1.example.org";
      tags = [ "web" ];
    };
    imports = [ ./hosts/web1 ];
  };

  web2 = {
    deployment = {
      targetHost = "web2.example.org";
      tags = [ "web" ];
    };
    imports = [ ./hosts/web2 ];
  };
};
```

```console
$ colmena apply --on @web
```

colmena ist explizit für **parallele** Deployments auf viele Hosts gebaut – `--on` erlaubt gezieltes Filtern über einzelne Hostnamen oder Tags (`@web` oben trifft beide Web-Server gleichzeitig). Als "stateless" bezeichnet sich das Projekt selbst, weil es außer dem, was ohnehin im Flake und im Store steht, keinen eigenen Zustand mitführt.

> ⚠️ **Vorsicht mit dem Default:** Ohne `--on` wendet `colmena apply` die Konfiguration standardmäßig auf **alle** definierten Hosts an – bei einem Setup mit mehreren Servern ein leicht zu übersehener Unterschied zum gewohnten Einzelmaschinen-Rebuild. Ein Tippfehler im Filter kann schlimmstenfalls die falsche Maschine treffen. Explizit filtern, gerade am Anfang.

## Vergleichstabelle: wann welches Tool

| Situation | Werkzeug |
|---|---|
| 1–2 Maschinen, kein Zusatztool gewünscht | `nixos-rebuild --target-host` |
| Maschine existiert noch nicht als NixOS (nur SSH/Rescue-Zugriff) | `nixos-anywhere` (+ `disko`) |
| Ausfallsicherheit beim Remote-Deploy wichtig (automatischer Rollback bei Fehlschlag) | deploy-rs |
| Viele gleichartige Hosts, parallel, tag-basiert gruppiert | colmena |

## Vollständiges Beispiel

Ein `nixos-anywhere`-Aufruf zur Erstprovisionierung, direkt gefolgt vom laufenden Update-Weg:

```console
# Erstprovisionierung einer frischen Maschine (nur SSH-Zugriff vorausgesetzt)
$ nix run github:nix-community/nixos-anywhere -- \
    --flake .#neuer-server root@192.168.1.50

# Ab jetzt läuft der Alltag über den eingebauten Weg
$ nixos-rebuild switch --flake .#neuer-server \
    --target-host root@192.168.1.50 --build-host localhost
```

> 💡 **Nice to know:** Terraform bzw. OpenTofu und Nix ergänzen sich gut, statt zu konkurrieren: Terraform/OpenTofu provisioniert die *Infrastruktur* (VM anlegen, Netzwerk, Firewall-Regeln beim Cloud-Anbieter), `nixos-anywhere`/colmena/deploy-rs übernehmen danach die *Betriebssystem*-Konfiguration on top. Eine saubere Trennung: "Existiert die Maschine?" ist Terraforms Frage, "Was läuft darauf?" ist Nix' Frage.

> 💡 **Nice to know:** `colmena apply --on @web` lässt sich problemlos aus einer CI-Pipeline heraus aufrufen (Forgejo Actions, GitHub Actions, …) – bei jedem Merge auf den Hauptbranch automatisch ausgerollt, im GitOps-Stil. Der SSH-Deploy-Key gehört dann als Secret in die CI-Konfiguration, nicht in dieselbe Konfiguration, die er deployt (Kapitel 10 lässt grüßen).

## Typische Fehler

**1. SSH-Zugriff auf die Zielmaschine fehlt:**

```
Permission denied (publickey).
```

*Ursache:* Der Public Key des deployenden Nutzers steht nicht in den `authorized_keys` der Zielmaschine, oder der falsche Nutzername wurde angegeben.
*Fix:* Key ergänzen (Kapitel 6) oder korrekten `--target-user`/`deployment.targetUser` setzen.

**2. Passwortloser sudo auf der Zielmaschine fehlt:**

```
sudo: a password is required
```

*Ursache:* Remote-Deploy-Tools laufen nicht-interaktiv und können kein Passwort eingeben, wenn `security.sudo.wheelNeedsPassword` nicht auf `false` steht.
*Fix:* Für einen *dedizierten* Deploy-Nutzer `security.sudo.wheelNeedsPassword = false;` setzen – oder direkt als root deployen, wo das vertretbar ist.

## Übung

1. Deploye deine Buch-VM-Konfiguration testweise auf eine zweite, frische VM über `nixos-rebuild --target-host` (SSH-Zugriff vorausgesetzt).
2. Richte ein minimales colmena-Setup mit deiner Buch-VM als einzigem Host ein und führe zunächst nur `colmena build` aus (nicht `apply`), um den Build-Teil isoliert zu testen.

**Lösungsskizze:**

Zu 1 und 2: Folge den Befehlsblöcken in den Abschnitten oben; bei Verbindungsproblemen zuerst die beiden "Typischen Fehler" dieses Kapitels gegenchecken.

## Zusammenfassung

- `nixos-rebuild --target-host` ist der werkzeuglose Weg für ein bis zwei Maschinen, ohne eingebaute Parallelisierung.
- `nixos-anywhere` ist ein Erstprovisionierungs-Werkzeug für bare Maschinen mit nur SSH-Zugriff, kombiniert mit disko.
- deploy-rs bietet mehrere Profile pro Node und einen automatischen "Magic Rollback" bei fehlgeschlagener Gesundheitsprüfung.
- colmena ist auf paralleles, tag-basiertes Deployment vieler Hosts ausgelegt – ohne `--on`-Filter trifft es standardmäßig alle definierten Hosts.
- Terraform/OpenTofu und Nix ergänzen sich: Infrastruktur vs. Betriebssystem-Konfiguration.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `nixos-rebuild --target-host/--build-host/--use-remote-sudo` | [Man-Page (mankier)](https://www.mankier.com/8/nixos-rebuild), [NixOS & Flakes Book – Remote Deployment](https://nixos-and-flakes.thiscute.world/best-practices/remote-deployment) |
| `security.sudo.wheelNeedsPassword` (Sicherheitshinweis) | [NixOS & Flakes Book – Remote Deployment](https://nixos-and-flakes.thiscute.world/best-practices/remote-deployment) |
| deploy-rs, Profile, CLI-Syntax | [deploy-rs – GitHub](https://GitHub.com/serokell/deploy-rs) |
| colmena, `deployment.targetHost`/`tags`, `--on`, "stateless" | [Colmena – GitHub](https://github.com/nix-community/colmena), [Colmena-Doku](https://colmena.cli.rs/0.3/examples/multi-arch.html), [BeloutreBlog – Deploying with Colmena](https://beloutreblog.yashael.fr/en/articles/deploying-nixos-configurations-with-colmena/) |
| `nixos-anywhere` | Community-Projekt, siehe Kapitel 4 |
| `Permission denied (publickey)`, `sudo: a password is required` | Standard-SSH-/sudo-Fehlermeldungen (allgemeines Linux-Wissen) |
