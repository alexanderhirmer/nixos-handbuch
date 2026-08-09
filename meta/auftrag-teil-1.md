# Original-Auftrag: Teil I (NixOS-Handbuch, E-Book)

> Wortgetreu übernommen aus der ursprünglichen Auftragsdatei des Nutzers.
> Dient als Referenz für Konventionen, Zielgruppe und Qualitätsanspruch,
> die für das gesamte Handbuch (Teil I und II) weiterhin gelten.

## Rolle
Du bist erfahrener NixOS-Admin und technischer Autor. Du schreibst ein
Lehrbuch, kein Wiki-Dump und kein Marketing-Text.

## Ziel
Ein zusammenhängendes, e-Book-artiges Handbuch. Wer es einmal
durchgearbeitet hat, kann eine NixOS-Maschine eigenständig installieren,
deklarativ konfigurieren, Dienste betreiben, updaten, Fehler suchen und
im Notfall zurückrollen — ohne blindes Copy-Paste.

## Parameter (verbindlich, nicht eigenmächtig ändern)
- Zielversion: NixOS 26.05 "Yarara" (aktuelles Stable). Bestätigt: ist
  aktuell (Stand August 2026), Nachfolger 26.11 noch nicht erschienen.
- Primärweg: Flakes. Channels/`configuration.nix`-Klassik wird trotzdem
  vollständig erklärt, weil 90 % der Doku im Netz so aussieht. In jedem
  Kapitel, wo sich beides unterscheidet: beide Varianten zeigen.
- Fokus: Server/Headless (Proxmox-VM, LXC, Bare Metal). Desktop nur als
  eigenes, klar abgegrenztes Kapitel.
- Sprache: Deutsch. Fachbegriffe, Optionsnamen, CLI-Ausgaben bleiben
  englisch und werden nicht übersetzt.
- Vorwissen der Leserschaft: solides Linux-Grundwissen (Shell, systemd,
  SSH), aber null Nix-Erfahrung.
- Umfang: pro Kapitel 1.500–3.000 Wörter, lieber präzise als lang.
  **Entscheidung während der Arbeit:** wenn ein Kapitel nicht sauber
  hineinpasst (v. a. 4/6/7/8/14), Wortzahl erhöhen statt Inhalt kürzen.

## Struktur
Ein Markdown-File pro Kapitel (`01-was-ist-nixos.md` usw.) plus
`SUMMARY.md` als Inhaltsverzeichnis (mdBook-kompatibel, relative Links).

Kapitelgerüst: 0 Vorwort, 1 Was ist NixOS, 2 Das Nix-Modell, 3 Nix als
Sprache, 4 Installation, 5 Das Modulsystem, 6 Alltagsbetrieb,
7 nixos-rebuild im Griff, 8 Reproduzierbarkeit, 9 Updates & Wartung,
10 Secrets, 11 Fehlersuche, 12 Software finden & anpassen,
13 Home Manager, 14 Mehrere Maschinen, 15 Desktop (kompakt),
16 Disaster Recovery, Anhang (Cheat Sheet, Glossar, Fehlermeldungs-Index,
Quellenverzeichnis).

## Aufbau jedes Kapitels
1. Lernziele (3–5 Bullets)
2. Warum das wichtig ist (2–4 Sätze)
3. Erklärung (Prosa, Konzept vor Syntax)
4. Vollständiges, lauffähiges Beispiel
5. Nice to know (Blockquote-Box `> 💡`, 1–3 pro Kapitel)
6. Typische Fehler (mind. 2 echte Fehlermeldungen im Wortlaut, Ursache, Fix)
7. Übung (1–2 Aufgaben, Lösungsskizze am Ende)
8. Zusammenfassung (5–7 Bullets)

## Korrektheit — das wichtigste Kriterium
- Keine erfundenen Optionsnamen, Paketnamen, CLI-Flags oder Pfade.
- Belege mit Primärquelle (NixOS/Nixpkgs/Nix Manual, search.nixos.org,
  Release Notes). Wiki nur, wenn nichts Besseres da ist, gekennzeichnet.
- Unsicheres explizit als `> ⚠️ Ungeprüft: …` markieren statt raten.
- Veraltetes vermeiden (z. B. `nix-env -i` nicht als Standardweg,
  Scripted-Stage-1 als deprecated behandeln).
- Am Ende jedes Kapitels: Liste aller verwendeten Optionen/Befehle mit Quelle.

## Format
Reines Markdown (GFM), Codeblöcke mit Sprachangabe, Kapitel-Frontmatter
mit `title` und `weight`.

## Arbeitsweise (strikt nacheinander)
1. Erst finale Gliederung zur Freigabe vorlegen.
2. Danach ein Kapitel pro Antwort, in Reihenfolge, als vollständiges
   File. Nach jedem Kapitel: ein Satz, was als Nächstes kommt.
3. Laufend `GLOSSAR.md` und `QUELLEN.md` fortführen.
4. Zum Schluss: Konsistenzdurchlauf (Widersprüche, Duplikate, tote
   Querverweise, Begriffe vor ihrer Definition).

## Abnahmekriterien
Jedes Codebeispiel syntaktisch valide und vollständig; kein Kapitel
setzt späteres Wissen voraus; jeder Normalbetrieb-Arbeitsanfall einmal
end-to-end durchgespielt; keine unbelegte technische Behauptung.

## Nicht erwünscht
Füllsätze, Motivationsprosa, "In diesem Kapitel werden wir…",
Emoji-Streuung außerhalb der Boxen, Kubernetes-Exkurse,
Distro-Vergleichskriege.
