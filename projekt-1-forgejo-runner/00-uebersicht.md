---
title: "Übersicht"
weight: 0
---

# Projekt 1: Gehärteter Forgejo-Runner-LXC

## Architekturüberblick

Ein einzelner LXC-Container auf Proxmox läuft als Forgejo-Actions-Runner: Er meldet sich bei einer bestehenden Forgejo-Instanz an und führt dort ausgelöste CI-Jobs aus. Login für reguläre Nutzer läuft über einen bestehenden LDAP-Server (per `sssd`, Passwort-Auth). Ein einzelner lokaler Nutzer außerhalb des LDAP – `f-local-admin-<hostname>` – dient als Fallback-Zugang für den Fall, dass LDAP mal nicht erreichbar ist: Login ausschließlich per SSH-Key, dafür passwortloses `sudo`. SSH läuft auf Port 40 statt 22, `fail2ban` blockt Brute-Force-Versuche, die Firewall lässt nur das Nötigste herein. Container-Anlage und alles, was sich als Datei ausdrücken lässt (Nutzer, Dienste, Firewall-Regeln), ist deklarativ – nichts wird interaktiv auf der Kommandozeile eingerichtet und dort vergessen.

Der Runner selbst braucht Zugriff auf eine `.env`-Datei mit Konfigurationswerten. In der Hauptvariante steht die im Klartext im Store (dafür ist sie nicht geheim – reine Konfiguration, keine Zugangsdaten); ein Exkurs am Ende zeigt, wie dieselbe Datei stattdessen sops-age-verschlüsselt aussähe.

## Ressourcenbedarf

- 1 LXC-Container (NixOS-Container-Image, kein Debian-Unterbau)
- 1 vCPU, 512 MB–1 GB RAM reichen für den Runner selbst; mehr, falls die auszuführenden Jobs selbst RAM-hungrig sind
- 4–8 GB Storage, mehr falls Docker-/Podman-Images für Job-Container lokal gecacht werden sollen
- Netzwerkzugriff auf einen bestehenden LDAP-Server und eine bestehende Forgejo-Instanz – beide werden vorausgesetzt, ihr Aufbau ist nicht Teil dieses Projekts

## Platzhaltertabelle

Diese Werte werden ab jetzt konsequent verwendet – ersetze sie 1:1 durch deine eigenen.

| Platzhalter | Bedeutung | Beispielwert |
|---|---|---|
| `<hostname>` | Hostname des Containers | `ci-runner-01` |
| `<ip>` | Interne IP-Adresse | `10.20.0.11` |
| `<ldap-uri>` | LDAP-Server-URI | `ldap://ldap.internal.example` |
| `<ldap-base-dn>` | LDAP Base DN | `dc=internal,dc=example` |
| `<forgejo-url>` | Basis-URL der Forgejo-Instanz | `https://git.internal.example` |
| `<runner-name>` | Name, unter dem sich der Runner bei Forgejo meldet | `ci-runner-01` |
| `<admin-user>` | Lokaler Fallback-Admin | `f-local-admin-ci-runner-01` |
| `<pve-storage>` | Proxmox-Storage-Backend für den Container | `local-lvm` |
| `<ssh-port>` | SSH-Port | `40` |
| `<vmid>` | Proxmox-Container-ID | `201` |
| `<repo-root>` | Wurzel deines eigenen NixOS-Infrastruktur-Repos (nicht dieses Buch-Repo) | `~/nixos-infra` |

Dateien für diese Maschine landen unter `<repo-root>/hosts/<hostname>/` – dieselbe Struktur wird in Projekt 3 auf mehrere Hosts erweitert.

## Was vorausgesetzt wird (nicht Teil dieses Projekts)

- Laufender Proxmox-VE-9.2-Host mit freier Kapazität
- Bestehende Forgejo-Instanz samt Netzwerkzugriff darauf
- Bestehender LDAP-Server mit den benötigten Nutzerkonten
- Netzwerk/Routing, über das `<hostname>` sowohl `<ldap-uri>` als auch `<forgejo-url>` erreicht
