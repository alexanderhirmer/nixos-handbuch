# NixOS-Handbuch

E-Book-artiges NixOS-Handbuch (Teil I) plus drei durchgängige
Praxisprojekte auf Proxmox (Teil II). Markdown, mdBook-kompatibel.

## Struktur

```
SUMMARY.md                       mdBook-Inhaltsverzeichnis (beide Teile)
GLOSSAR.md                       laufendes Glossar
QUELLEN.md                       laufendes Quellenverzeichnis
00-vorwort.md … 17-anhang.md     Teil I, vollständig (17 Kapitel + Anhang)
projekt-1-forgejo-runner/        Teil II, Projekt 1 (in Arbeit)
projekt-2-vaultwarden/           Teil II, Projekt 2 (noch nicht begonnen)
projekt-3-fleet/                 Teil II, Projekt 3 (noch nicht begonnen)
meta/                            Aufträge, Entscheidungen, Status — siehe unten
```

## Für die Fortsetzung mit Claude / Claude Code

**Zuerst lesen, in dieser Reihenfolge:**

1. `meta/PROZESS.md` — wie hier gearbeitet wird, Vorgehen pro Schritt
2. `meta/ENTSCHEIDUNGEN.md` — alle bisherigen Entscheidungen, Konventionen, Platzhalterwerte, recherchierte technische Fakten
3. `meta/status-teil-2.md` — vollständige, freigegebene Schrittpläne aller drei Projekte inkl. Fortschritt
4. `meta/auftrag-teil-1.md` / `meta/auftrag-teil-2.md` — die ursprünglichen Aufträge im Wortlaut, falls Details in den drei Dateien oben nicht reichen

Teil I ist abgeschlossen und durchlief bereits einen Konsistenzdurchlauf
— nicht verändern, nur referenzieren (Querverweise "Kapitel N" aus
Teil II zeigen dorthin).

Teil II wird Schritt für Schritt fortgesetzt: ein Schritt pro Antwort,
im Achtteil-Format aus `meta/auftrag-teil-2.md`, mit Recherche vor
jeder neuen technischen Behauptung. Nach jedem geschriebenen Schritt
`meta/status-teil-2.md` aktualisieren (Häkchen setzen, `SUMMARY.md`
ergänzen).

## Bauen (mdBook)

```console
$ cargo install mdbook   # falls noch nicht installiert
$ mdbook build
$ mdbook serve           # lokale Vorschau
```
