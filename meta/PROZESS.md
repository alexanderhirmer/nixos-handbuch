# Prozess: Wie dieses Handbuch entsteht

Für eine neue Claude-Instanz (z. B. Claude Code), die hier weiterarbeitet.
Ziel: gleiche Qualität und Vorgehensweise, ohne die vorherige Unterhaltung
gesehen zu haben.

## Grundprinzip

Alles, was für die Fortsetzung gebraucht wird, steht in diesem Repo —
nicht im Gedächtnis eines Chat-Tools. Das `meta/`-Verzeichnis ist die
Quelle der Wahrheit für Vorgaben, Entscheidungen und offene Punkte.
Vor dem Weiterschreiben immer zuerst `meta/ENTSCHEIDUNGEN.md` und die
für Teil II relevante `meta/status-teil-2.md` lesen.

## Teil I — wie es entstanden ist

1. Auftrag (`meta/auftrag-teil-1.md`) gelesen, NixOS-Version verifiziert
   (Websuche: 26.05 "Yarara" ist aktuelle Stable).
2. Vollständige Gliederung (17 Kapitel + Anhang) vorgeschlagen, dabei
   Reibungspunkte offen benannt (v. a. Wortbudget vs. Anspruch bei
   Channels+Flakes+drei Zielplattformen) — Freigabe eingeholt.
3. Ein Kapitel pro Antwort geschrieben, strikt nach dem in
   `auftrag-teil-1.md` vorgegebenen Achtteil-Aufbau. Vor technisch
   heiklen Passagen (Modulsystem-Mechanik, Flake-Syntax, aktuelle
   Options-Umbenennungen wie `services.xserver.displayManager.*` →
   `services.displayManager.*`) gezielt recherchiert statt aus dem
   Gedächtnis geschrieben.
4. `GLOSSAR.md` und `QUELLEN.md` nach jedem Kapitel um die neuen
   Begriffe/Quellen ergänzt (Datei lesen, Diff anhängen, nicht
   überschreiben).
5. Am Ende: Konsistenzdurchlauf per `grep` über alle Kapitel-
   Querverweise ("Kapitel N"), Glossar-Duplikate, Begriffs-
   Erstverwendung vor Definition. Gefundene Probleme direkt behoben
   (Beispiele: `services.sshd.enable` vs. `services.openssh.enable`
   vereinheitlicht; fehlender `services.xserver.xkb.layout`-Abschnitt
   in Kapitel 15 nachgetragen, den Kapitel 6 versprochen hatte;
   `system.stateVersion` in Kapitel 4 ergänzt, weil Kapitel 8 es
   voraussetzte, ohne dass es je eingeführt wurde).

## Teil II — wie es begonnen hat

Vier-Phasen-Modell aus `auftrag-teil-2.md` (siehe dort) eingehalten:
Phase 0 (Fragen) → Phase 1 (Exposés, Freigabe) → Phase 2 (Schrittpläne,
Freigabe) → Phase 3 (ein Schritt pro Antwort). Die Antworten aus Phase 0
und die Exposés/Schrittpläne aus Phase 1/2 stehen vollständig in
`auftrag-teil-2.md` bzw. `status-teil-2.md` — nicht erneut nachfragen,
sondern von dort übernehmen.

Für Projekt 2 (Vaultwarden) und Projekt 3 (Fleet) hat die KI die
Hauptthemen selbst vorgeschlagen (Nutzer wollte dort nur Nebenrollen
vorgeben) — Begründung und exakter Wortlaut der Exposés stehen in
`status-teil-2.md`.

## Wiederkehrende Arbeitsweise pro Schritt (Teil II, Phase 3)

1. Vorherigen Schritt und `status-teil-2.md` lesen, um Anschluss zu
   finden (Platzhalterwerte, zuletzt erreichter Zustand).
2. Bei neuen technischen Behauptungen (Optionsnamen, CLI-Syntax,
   GUI-Menüpfade) gezielt recherchieren, bevor geschrieben wird —
   siehe Korrektheitsregeln in `auftrag-teil-1.md`/`auftrag-teil-2.md`.
   Beispiele aus bisherigen Schritten: `pct create`-Syntax,
   `--ostype unmanaged` für NixOS-Container, `services.gitea-actions-
   runner`, dass eigene Proxmox-Templates über
   `/var/lib/vz/template/cache/` statt `pveam add` eingespielt werden.
3. Schritt exakt im Achtteil-Format aus `auftrag-teil-2.md` schreiben
   (Ziel/Voraussetzung/Durchführung/Dateien/Prüfen/Wenn's schiefgeht/
   Rückweg/Querverweis), 300–700 Wörter.
4. Datei unter `projekt-N-<slug>/NN-<slug>.md` anlegen, `SUMMARY.md`
   um den Eintrag ergänzen.
5. `status-teil-2.md` aktualisieren: Schritt als erledigt markieren,
   ggf. neue Platzhalter oder Konventionen eintragen, die künftige
   Schritte betreffen.
6. Ein Satz, was als Nächstes kommt — dann stoppen und auf Freigabe/
   Fortsetzung warten.

## Korrektheitsdisziplin (für beide Teile bindend)

- Nichts erfinden: Optionsnamen, Paketnamen, CLI-Flags, GUI-Pfade nur
  nach Verifikation (Websuche gegen Primärquelle) verwenden.
- Unsicheres explizit mit `> ⚠️ Ungeprüft: …` kennzeichnen statt zu
  glätten.
- Reale, zitierfähige Fehlermeldungen verwenden (aus Issues, offizieller
  Doku, Foren) statt plausibel klingende zu erfinden.
- Bei Versionsabhängigkeit (z. B. sich änderndes CLI-Verhalten bei
  Community-Tools wie disko) das explizit benennen statt eine Version
  als ewig gültig hinzustellen.
