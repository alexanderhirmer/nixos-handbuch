# Original-Auftrag: Teil II (Drei Praxisprojekte)

> Wortgetreu übernommen aus der ursprünglichen Auftragsdatei des Nutzers.

## Kontext
Teil I ist fertig. Teil II baut darauf auf und wiederholt nichts daraus.

## Ziel
Drei durchgängige Praxisprojekte mit steigender Komplexität. Jedes
Projekt führt vom leeren Proxmox-Host (PVE 9.2, Debian 13 "Trixie")
bis zum lauffähigen Endzustand — Befehl für Befehl, Datei für Datei.
Wer Teil I gelesen hat, muss jedes Projekt ohne weitere Recherche
nachbauen können.

## Arbeitsweise (vier Phasen, strikt nacheinander)
- **Phase 0:** Erst fragen, dann planen. Kompakter Fragenblock, bevor
  irgendetwas geschrieben wird.
- **Phase 1:** Pro Projekt ein Exposé (max. 200 Wörter): Endzustand,
  Stack, Lernziele, Neuerung ggü. Vorprojekt, geschätzte Schrittzahl,
  Ressourcenbedarf. Freigabe abwarten.
- **Phase 2:** Pro Projekt eine nummerierte Schrittliste (Richtwert
  15–30 Schritte), je ein Satz zum Ergebnis. Freigabe abwarten.
- **Phase 3:** Schreiben, ein Schritt pro Antwort. Richtwert
  300–700 Wörter pro Schritt. Höchstens eine neue Konfigurationsdatei
  oder ein zusammengehöriger Befehlsblock pro Schritt. Nach jedem
  Schritt ein Satz, was als Nächstes kommt, dann stoppen.

Komplexitätsstaffel:
- **Projekt 1:** ein Host, ein nützlicher Dienst, alles nachvollziehbar.
- **Projekt 2:** mehrere Hosts/Dienste, Secrets, Remote-Deployment,
  Reverse Proxy mit TLS, Backup und Restore.
- **Projekt 3:** reproduzierbare Flotte — Repo-Struktur für mehrere
  Maschinen, automatisierte Bereitstellung (disko/nixos-anywhere), CI,
  Monitoring, geprobter Disaster-Recovery-Fall.

## Aufbau jedes Schritts
1. Ziel (ein Satz)
2. Voraussetzung (welcher Schritt muss abgeschlossen sein)
3. Durchführung — exakte Befehle. Wo Proxmox-GUI nötig ist: Menüpfad
   mit Versionsangabe **plus** CLI-Äquivalent (`qm`, `pct`, `pvesm`,
   `pvesh`). CLI ist maßgeblich, GUI die Zugabe.
4. Dateien — vollständig, absoluter Pfad, Position im Repo. Änderungen
   an Bestandsdateien im Diff-Stil mit Kontext.
5. Prüfen — überprüfbares Erfolgskriterium. Keine erfundenen
   Terminal-Ausgaben; beschreiben, was zu sehen sein muss.
6. Wenn's schiefgeht — 1–3 realistische Fehlerbilder, Ursache, Fix.
7. Rückweg — wie man den Schritt rückgängig macht.
8. Querverweis — auf das Teil-I-Kapitel, wo das Konzept erklärt ist.

## Projektanfang und -abschluss
- Zu Beginn: Architekturüberblick, Ressourcenbedarf, Platzhalter-Tabelle
  (Hostnamen, IPs, Domains, Nutzer) — danach konsequent verwendet.
- Am Ende: vollständiger Verzeichnisbaum, alle Endkonfigurationen
  gesammelt, "Was du jetzt kannst", 3–5 Ausbaustufen, Teardown.

## Korrektheit — weiterhin oberste Priorität
Keine erfundenen Optionsnamen/Pakete/Flags/Pfade; alles muss in NixOS
26.05 bzw. PVE 9.2 existieren. Primärquellen belegen, Unsicheres als
`> ⚠️ Ungeprüft: …` markieren — besonders bei GUI-Menüpfaden und
Ausgabetexten. Alle externen Abhängigkeiten gepinnt, keine `latest`-
Verweise.

## Format & Integration
Ordner je Projekt: `projekt-N-<slug>/`, darin `00-uebersicht.md`,
`01-<slug>.md`, … Ein File pro Schritt, Frontmatter mit `title` und
`weight`. `SUMMARY.md` erweitern, `GLOSSAR.md`/`QUELLEN.md` weiterpflegen.

## Nicht erwünscht
Wiederholung von Teil-I-Erklärungen, Motivationsprosa, Schritte ohne
Prüfkriterium, Platzhalter, die nicht in der Tabelle stehen.

---

## Klärungsrunde (Phase 0) — Antworten des Nutzers

1. **Projekt 1, Endzustand/Stack:** Ein LXC-Server als Forgejo-Runner
   (Paket existiert bereits: `services.gitea-actions-runner`,
   `pkgs.forgejo-runner`). Gehärtet. `.env`-Datei, auf die der Runner
   zugreift (zunächst ohne Secrets, Exkurs mit Secrets gewünscht).
   Nutzer kommen aus LDAP. Hostname konfigurierbar. Ein lokaler
   Nicht-LDAP-User `f-local-admin-<hostname>` darf passwortlos sudo,
   loggt sich aber **nur per SSH-Key** ein (kein Passwort-Login).
   Andere Nutzer sudoen nicht passwortlos. SSH-Login ganz ohne
   Passwort ist ansonsten nicht erlaubt. SSH-Port wird auf 40 verlegt;
   ob `/etc/services` unter NixOS angepasst werden muss, war zu
   recherchieren (Ergebnis: nein, siehe `ENTSCHEIDUNGEN.md`).
   `fail2ban` gehört zur Härtung dazu.
2. **Komplexitätsstufe Wunschprojekt:** Stufe 1.
3. **Punkte für Projekt 2/3 (Nebenrollen, nicht Hauptthema):** eigenes
   SSH-Server-Zertifikat von privater Root-CA signiert; Secrets mit
   sops-age verschlüsselt; GitOps mit automatischer Anwendung
   geänderter Git-Manifeste. Hauptthemen durfte die KI selbst wählen.
4. **Ressourcen:** spielt keine Rolle, keine Vorgabe.
5. **Bestehende Infrastruktur** (Netz/VLANs, DNS, Domain, CA,
   Secrets-Setup, Git-Hosting): gilt als gegeben, muss in den
   Projekten nicht erklärt/aufgebaut werden.
6. **Netzwerk:** rein internes Netz, kein öffentlicher Zugriff/ACME.
7. **Ausgeschlossene Technologien:** keine.

Zusätzliche Vorgaben (nach Phase 1):
- Festplatten/Partitionen **immer** deklarativ in einer Datei
  konfiguriert, nichts per Ad-hoc-Befehl (bei LXC: `pct create`-
  Parameter fest in einem versionierten Skript statt disko, da LXC
  keine eigene Partitionierung hat — echtes disko in Projekt 2/3, die
  auf VMs laufen).
- Projekte 2 und 3 dürfen auf Schritte aus Vorprojekten verweisen und
  diese übernehmen, nur Unterschiede erklären — spart Doppelarbeit.
