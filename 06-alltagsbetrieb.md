---
title: "Alltagsbetrieb"
weight: 6
---

# Alltagsbetrieb

## Lernziele

- Du kannst Pakete systemweit, nutzerspezifisch und ad-hoc installieren und weißt, wann welcher Weg passt.
- Du aktivierst Dienste über das immer gleiche `services.<name>.enable`-Muster.
- Du verwaltest Nutzer und SSH-Keys vollständig deklarativ.
- Du konfigurierst Netzwerk-Grundlagen und die Firewall.
- Du setzt Locale, Zeitzone und Tastaturlayout.
- Du definierst eine eigene systemd-Unit deklarativ.

## Warum das wichtig ist

Das hier ist der Alltag: Der Großteil dessen, was du nach der Installation an einer Maschine tust, fällt in dieses Kapitel. Die gute Nachricht: Fast alles davon ist reines `config` (Kapitel 5) für Optionen, die Nixpkgs längst deklariert hat – du musst selten eigene Module schreiben, nur die richtigen Optionen kennen.

## Pakete: drei Wege

**Systemweit**, für alle Nutzer, braucht einen Rebuild:

```nix
{ environment.systemPackages = [ pkgs.thunderbird pkgs.emacs ]; }
```

**Nutzerspezifisch**, ohne Rebuild, nur für einen Nutzer sichtbar:

```console
$ nix-env -iA nixos.thunderbird
```

Landet – als root ausgeführt – im systemweiten Default-Profil, sonst im Profil des jeweiligen Nutzers (`/nix/var/nix/profiles/per-user/<name>/profile`, siehe Kapitel 2). Sauberer als `nix-env` ist meist Home Manager (Kapitel 13), aber das ist ein eigenes Projekt, keine NixOS-Kernfunktion.

**Ad-hoc**, temporär, nur für die aktuelle Shell-Sitzung, nichts wird dauerhaft "installiert":

```console
$ nix shell nixpkgs#hello
$ hello
Hello, world!
```

Verlässt du die Shell, ist `hello` wieder weg – nichts wurde in ein Profil geschrieben. Das braucht die (formal weiterhin experimentellen, praktisch aber breit genutzten) Nix-Command/Flakes-Features; falls `nix shell` mit einem Hinweis auf fehlende experimentelle Features abbricht, hilft `nix.settings.experimental-features = [ "nix-command" "flakes" ];` in der Konfiguration.

## Dienste aktivieren

Das wiederkehrende Muster: `services.<name>.enable = true;`, optional ergänzt um Feinkonfiguration. Am Beispiel SSH, inklusive ein paar gängiger `settings`:

```nix
services.openssh = {
  enable = true;
  openFirewall = true;
  settings = {
    PasswordAuthentication = false;
    PermitRootLogin = "no";
  };
};
```

(`services.sshd.enable` funktioniert übrigens auch – es ist laut NixOS-Optionsdatenbank ein Alias auf `services.openssh.enable` – aber `services.openssh` ist der Name, unter dem auch alle weiteren Unteroptionen wie `settings` und `openFirewall` hängen, deshalb konsequent dieser Name in diesem Buch.)

## Nutzerverwaltung & SSH-Keys

Deklarativ, wie schon in Kapitel 4 kurz gesehen, hier vollständiger:

```nix
users.users.alex = {
  isNormalUser = true;
  description = "Alex";
  extraGroups = [ "wheel" "networkmanager" ];
  openssh.authorizedKeys.keys = [
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... alex@laptop"
  ];
};
```

`wheel` erlaubt `sudo`, `networkmanager` erlaubt eigenständige Netzwerkkonfiguration. Nutzer, die so angelegt werden, haben zunächst kein Passwort – für interaktives Login brauchst du zusätzlich `passwd alex` (oder ein deklaratives `hashedPassword`, erzeugt z. B. mit `mkpasswd`).

Mit `users.mutableUsers = false;` wird das strikt: `/etc/passwd` und `/etc/group` entsprechen dann exakt der Konfiguration, und imperative Tools wie `useradd` oder `passwd` funktionieren gar nicht mehr – jede Änderung muss über `configuration.nix` und einen Rebuild laufen.

## Netzwerk-Grundlagen & Firewall

```nix
networking.hostName = "buch-vm";
networking.firewall.enable = true;          # ist ohnehin die Voreinstellung
networking.firewall.allowedTCPPorts = [ 80 443 ];
networking.firewall.allowPing = false;
```

Die NixOS-Firewall ist standardmäßig aktiv und blockt eingehende Verbindungen, bis du Ports explizit freigibst. Viele Dienst-Module – wie oben bei `services.openssh.openFirewall` gesehen – bringen dafür eine bequeme eigene Option mit, statt dass du den Port manuell in `allowedTCPPorts` nachtragen musst.

## Locale, Zeitzone, Tastaturlayout

Drei kleine, voneinander unabhängige Optionen:

```nix
time.timeZone = "Europe/Berlin";
i18n.defaultLocale = "de_DE.UTF-8";
console.keyMap = "de";
```

`console.keyMap` gilt für die textbasierte Konsole (also praktisch immer, auf einem Server). Für eine grafische Oberfläche kommt zusätzlich `services.xserver.xkb.layout` ins Spiel – relevant erst ab Kapitel 15, auf einer reinen Server-Maschine brauchst du das in aller Regel nicht.

## Eigene systemd-Units deklarativ definieren

```nix
systemd.services.book-heartbeat = {
  description = "Schreibt bei jedem Boot einen Zeitstempel ins Log";
  wantedBy = [ "multi-user.target" ];
  serviceConfig.Type = "oneshot";
  script = ''
    ${pkgs.coreutils}/bin/date >> /var/log/book-heartbeat.log
  '';
};
```

Zwei Details, die typisch für NixOS sind: `wantedBy = [ "multi-user.target" ]` sorgt dafür, dass die Unit beim normalen Hochfahren mitgestartet wird, und `${pkgs.coreutils}/bin/date` referenziert einen konkreten Store-Pfad statt sich auf `$PATH` zu verlassen – reproduzierbar, unabhängig davon, was sonst noch installiert ist.

## Vollständiges Beispiel

Alles aus diesem Kapitel in einer Konfiguration:

```nix
{ config, pkgs, ... }:
{
  networking.hostName = "buch-vm";

  time.timeZone = "Europe/Berlin";
  i18n.defaultLocale = "de_DE.UTF-8";
  console.keyMap = "de";

  services.openssh = {
    enable = true;
    openFirewall = true;
    settings.PasswordAuthentication = false;
  };

  networking.firewall.allowedTCPPorts = [ 80 ];

  users.users.alex = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... alex@laptop"
    ];
  };

  systemd.services.book-heartbeat = {
    description = "Schreibt bei jedem Boot einen Zeitstempel ins Log";
    wantedBy = [ "multi-user.target" ];
    serviceConfig.Type = "oneshot";
    script = ''
      ${pkgs.coreutils}/bin/date >> /var/log/book-heartbeat.log
    '';
  };

  environment.systemPackages = [ pkgs.htop ];
}
```

Nach einem Rebuild (Details Kapitel 7): `ssh alex@<ip-der-vm>` sollte funktionieren, `cat /var/log/book-heartbeat.log` zeigt den Zeitstempel vom letzten Boot.

> 💡 **Nice to know:** Bewusst NICHT in diesem Kapitel: irgendetwas, das ein echtes Geheimnis ist (API-Token, private Zertifikate). SSH-*öffentliche* Schlüssel und gehashte Passwörter sind unkritisch direkt in `configuration.nix` – aber der Nix Store ist world-readable (Kapitel 2!), und alles, was dort für jeden lesbar landen würde, gehört nicht hinein. Der saubere Weg dafür ist Kapitel 10.

> 💡 **Nice to know:** `nixos-option` (schon aus Kapitel 5 bekannt) ist auch hier dein Freund, um nachzuschauen, was ein Dienst gerade tatsächlich für einen Wert hat – gerade wenn mehrere Module denselben Bereich berühren.

## Typische Fehler

**1. SSH-Verbindung wird abgelehnt:**

```
ssh: connect to host 192.168.1.50 port 22: Connection refused
```

*Ursache:* Entweder läuft `services.openssh` gar nicht, oder die Firewall blockt Port 22, weil `openFirewall` fehlt und der Port nicht manuell freigegeben wurde.
*Fix:* `services.openssh.enable = true;` und `services.openssh.openFirewall = true;` setzen (oder den Port manuell in `networking.firewall.allowedTCPPorts` eintragen), dann rebuilden.

**2. `useradd` funktioniert nicht mehr:**

```
useradd: cannot lock /etc/passwd; try again later.
```

*Ursache:* `users.mutableUsers = false;` ist gesetzt – `/etc/passwd` und `/etc/group` sind dann strikt an die Konfiguration gekoppelt, imperative Tools greifen ins Leere.
*Fix:* Den Nutzer stattdessen deklarativ unter `users.users.<name>` anlegen und rebuilden.

## Übung

1. Füge einen zweiten Nutzer zu deiner Buch-VM hinzu, mit eigenem SSH-Key und Mitgliedschaft in `wheel`. Rebuild, dann per SSH einloggen (Port ggf. freigeben).
2. Übernimm die `book-heartbeat`-Unit aus dem Kapitel-Beispiel, rebuilde, starte neu und prüfe mit `systemctl status book-heartbeat` und `cat /var/log/book-heartbeat.log`, dass sie gelaufen ist.

**Lösungsskizze:**

Zu 1 und 2: Folge den Beispielen oben in Reihenfolge; bei Verbindungsproblemen zuerst den ersten "Typischen Fehler" dieses Kapitels gegenchecken.

## Zusammenfassung

- Drei Wege für Pakete – systemweit, nutzerspezifisch, ad-hoc – unterscheiden sich vor allem in Persistenz und Sichtbarkeit.
- Dienste folgen fast immer demselben Muster: `services.<name>.enable = true;` plus optionale `settings`.
- Nutzer und SSH-Keys lassen sich komplett deklarativ abbilden; `mutableUsers = false` macht das strikt.
- Die Firewall ist standardmäßig aktiv; viele Dienst-Module bringen ein bequemes `openFirewall` mit.
- Locale, Zeitzone und Tastaturlayout sind drei kleine, unabhängige Optionen.
- Eigene systemd-Units lassen sich vollständig deklarativ definieren, inklusive reproduzierbarer Store-Pfade statt `$PATH`.
- Echte Secrets gehören nicht in dieses Kapitel – dafür ist Kapitel 10 da.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `environment.systemPackages`, `nix-env -iA` | [NixOS Manual – Package Management](https://nixos.org/manual/nixos/stable/) |
| `nix shell` | Nix Reference Manual (nix-command/flakes) |
| `services.openssh` (inkl. `settings`, `openFirewall`), `services.sshd.enable` als Alias | [NixOS-Wiki – SSH](https://wiki.nixos.org/wiki/SSH), [MyNixOS – services.sshd.enable](https://mynixos.com/nixpkgs/option/services.sshd.enable) |
| `users.users.<name>`, `users.mutableUsers` | [NixOS Manual – User Management](https://nixos.org/manual/nixos/stable/) |
| `networking.firewall.*` | [NixOS Manual – Configuration Syntax](https://nixos.org/manual/nixos/stable/) |
| `time.timeZone`, `i18n.defaultLocale`, `console.keyMap` | Nixpkgs-Quellcode (`nixos/modules/config/i18n.nix`), NixOS-Wiki – Locales |
| `systemd.services.<name>` (`script`, `serviceConfig.Type`, `wantedBy`) | Nixpkgs-Praxisbeispiel (u. a. `nixos-generators`-Issue-Tracker) |
| `useradd: cannot lock /etc/passwd` | Standard-shadow-utils-Fehlermeldung (allgemeines Linux-Wissen) |
