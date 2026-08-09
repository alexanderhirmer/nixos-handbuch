---
title: "Reproduzierbarkeit"
weight: 8
---

# Reproduzierbarkeit

## Lernziele

- Du verstehst, wie Channels funktionieren und wo ihre Reproduzierbarkeits-Grenzen liegen.
- Du kannst ein `flake.nix` lesen und schreiben (`description`, `inputs`, `outputs`).
- Du weißt, was `flake.lock` genau festhält und wann du es aktualisierst.
- Du kannst mit `nix flake update` gezielt einzelne oder alle Inputs aktualisieren.
- Du kannst eine bestehende Channel-Konfiguration zu Flakes migrieren.
- Du kennst eine Alternative, um Nixpkgs auch ohne Flakes zu pinnen.

## Warum das wichtig ist

Reproduzierbarkeit ist NixOS' zentrales Versprechen (Kapitel 1) – aber Channels und Flakes lösen das unterschiedlich gut. Wer den Unterschied nicht kennt, wundert sich irgendwann, warum "dieselbe" Konfiguration auf zwei Maschinen zu unterschiedlichen Softwareständen führt.

## Channels: wie sie funktionieren

Ein Channel ist ein rollender Zeiger auf einen bestimmten Nixpkgs-Stand, verwaltet über `nix-channel`:

```console
# nix-channel --add https://channels.nixos.org/nixos-26.05 nixos
# nix-channel --update
```

Stabile Channels (`nixos-26.05`) bekommen nur konservative Bugfixes; der `nixos-unstable`-Channel folgt direkt der Entwicklung; `nixos-26.05-small` enthält weniger vorgebaute Binärpakete, dafür schneller aktualisiert – praktisch für Server ohne GUI-Anspruch.

**Die Grenze der Reproduzierbarkeit:** Ein Channel-Name wie `nixos-26.05` zeigt auf "den aktuellen Stand dieses Zweigs" – läuft `nix-channel --update` auf zwei Maschinen zu unterschiedlichen Zeitpunkten, bekommen sie unterschiedliche Nixpkgs-Revisionen, obwohl beide denselben Channel-Namen benutzen. Es gibt keine eingebaute, versionierbare Datei, die exakt festhält, welche Revision gerade aktiv war – das musst du dir selbst merken (`nix-channel --list` plus Datum, wenn überhaupt).

## Flakes: `flake.nix`-Aufbau

Ein minimales `flake.nix` für eine NixOS-Konfiguration:

```nix
{
  description = "Meine Buch-VM";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  };

  outputs = { self, nixpkgs, ... }@inputs: {
    nixosConfigurations.buch-vm = nixpkgs.lib.nixosSystem {
      modules = [ ./configuration.nix ];
    };
  };
}
```

Drei Teile:

- **`description`** – reiner Klartext, keine funktionale Bedeutung.
- **`inputs`** – woher kommt was. Jeder Input hat eine `url` (hier ein GitHub-Branch); Inputs können auch von anderen Inputs abhängen.
- **`outputs`** – eine Funktion (genau das Konzept aus Kapitel 3!), die die Inputs als Argumente bekommt und die eigentlichen Ergebnisse zurückgibt – hier `nixosConfigurations.buch-vm`, aufgebaut mit `nixpkgs.lib.nixosSystem`.

`system = "x86_64-linux";` lässt sich in aktuellen Nixpkgs-Versionen meist weglassen, weil die generierte `hardware-configuration.nix` diese Information ohnehin schon mitbringt.

Willst du Inputs von Inputs erzwingen (z. B. damit Home Manager garantiert dieselbe Nixpkgs-Revision nutzt wie dein System), gibt es `follows`:

```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  home-manager.url = "github:nix-community/home-manager";
  home-manager.inputs.nixpkgs.follows = "nixpkgs";
};
```

## `flake.lock`: was gepinnt wird

Sobald du zum ersten Mal etwas mit deinem Flake machst (`nix flake show`, `nixos-rebuild switch --flake`, …), entsteht automatisch ein `flake.lock` – eine JSON-Datei, die für *jeden* Input exakt festhält: den Git-Revision-Hash (`rev`) und den Inhalts-Hash (`narHash` – derselbe Hash-Gedanke wie bei Store-Pfaden aus Kapitel 2, nur auf den Input als Ganzes angewendet). Genau das fehlt Channels: eine explizite, versionskontrollierbare Datei, die exakt fixiert, was "gerade aktiv" war. `flake.lock` gehört mit ins Git-Repo – erst dann ist die Konfiguration wirklich reproduzierbar, auf jeder Maschine, zu jedem Zeitpunkt.

## `nix flake update` im Detail

```console
$ nix flake update
```

aktualisiert **alle** Inputs auf den jeweils neuesten Stand entlang der in `inputs.*.url` angegebenen Branches/Referenzen. Willst du gezielt nur einen Input anfassen:

```console
$ nix flake update nixpkgs
```

(Ältere Nix-Versionen kennen dafür die Syntax `nix flake lock --update-input nixpkgs` – beide Formen tauchen in freier Wildbahn auf; welche bei dir funktioniert, hängt von der Nix-Version ab.) Das Ergebnis ist in beiden Fällen dasselbe: `flake.lock` ändert sich, `flake.nix` nicht. Ein `git diff flake.lock` danach zeigt dir genau, welche Revision durch welche ersetzt wurde.

## Migrationspfad: Channels → Flakes

1. `system.stateVersion` in deiner bestehenden `configuration.nix` nachsehen (jede installierte NixOS-Maschine hat das).
2. `flake.nix` anlegen, das denselben (oder einen neueren) Nixpkgs-Branch referenziert und die *bestehende* `configuration.nix` unverändert importiert:

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";

  outputs = { self, nixpkgs, ... }: {
    nixosConfigurations.nixos = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [ ./configuration.nix ];
    };
  };
}
```

3. `sudo nixos-rebuild switch --flake '/etc/nixos#nixos'` ausführen. Richtig gemacht, ändert sich an deinem laufenden System **nichts** – das ist der Beweis, dass die Migration sauber war, nicht ein Zeichen, dass nichts passiert ist.
4. Ab jetzt `nix-channel` links liegen lassen; Updates laufen nur noch über `nix flake update` plus Rebuild.

## Nixpkgs pinnen ohne Flakes

Auch ganz ohne Flakes lässt sich eine feste Nixpkgs-Revision referenzieren, mit `builtins.fetchTarball`:

```nix
let
  pinnedNixpkgs = builtins.fetchTarball {
    url = "https://github.com/NixOS/nixpkgs/archive/<commit-hash>.tar.gz";
    sha256 = "<hier gehört der echte Hash hin, z. B. via `nix-prefetch-url --unpack`>";
  };
in
import pinnedNixpkgs { }
```

Das ist weniger komfortabel als `flake.lock` (kein automatisches Update-Tooling, Hash musst du selbst ermitteln), aber es funktioniert ganz ohne experimentelle Features und ist für Nutzer, die (noch) nicht auf Flakes wechseln wollen, die gebräuchlichste Alternative zu einem rollenden Channel.

## Vollständiges Beispiel

Ein Ausschnitt aus einer echten `flake.lock` (Struktur, keine echten Hash-Werte):

```json
{
  "nodes": {
    "nixpkgs": {
      "locked": {
        "lastModified": 1754000000,
        "narHash": "sha256-AAAA...",
        "owner": "NixOS",
        "repo": "nixpkgs",
        "rev": "abc123...",
        "type": "github"
      },
      "original": {
        "owner": "NixOS",
        "ref": "nixos-26.05",
        "repo": "nixpkgs",
        "type": "github"
      }
    },
    "root": { "inputs": { "nixpkgs": "nixpkgs" } }
  },
  "root": "root",
  "version": 7
}
```

`original` ist das, was du in `flake.nix` geschrieben hast (der bewegliche Branch); `locked` ist der exakt fixierte Stand. Genau dieser Unterschied ist der Kern von Flakes' Reproduzierbarkeits-Versprechen.

> 💡 **Nice to know:** Flakes sind trotz ihres De-facto-Standard-Status – so gut wie jedes neuere Nixpkgs-Tutorial nutzt sie – formal weiterhin ein experimentelles Feature (`nix-command` und `flakes` müssen explizit aktiviert werden). Das ist keine Ankündigung baldiger Instabilität, aber ein guter Grund, in eigenen Projekten kurz zu dokumentieren, welche Nix-Version vorausgesetzt wird.

> 💡 **Nice to know:** `nix flake init -t <template>` legt ein neues Flake aus einer Vorlage an, z. B. `nix flake init -t github:nix-community/templates`. Für den Einstieg in eigene, projektspezifische Dev-Umgebungen (nicht nur NixOS-Systeme) ein guter Startpunkt, um nicht bei null anzufangen.

## Typische Fehler

**1. Uncommittete Änderungen im Flake-Repo:**

```
warning: Git tree '/etc/nixos' is dirty
```

*Ursache:* Flakes ziehen ihren Inhalt aus dem Git-Zustand; uncommittete Änderungen werden zwar meist trotzdem verwendet, aber eben *nicht* Teil dessen, was `flake.lock` als reproduzierbaren Stand versteht.
*Fix:* Änderungen committen (`git add`, `git commit`), bevor du dich auf Reproduzierbarkeit verlässt.

**2. Neuen Input in `inputs` ergänzt, aber nicht in der `outputs`-Funktionssignatur:**

```
error: undefined variable 'home-manager'
```

*Ursache:* `outputs` ist – wie in Kapitel 3 gelernt – schlicht eine Funktion; ein Input, der im Funktionsrumpf verwendet werden soll, muss im Parameter-Pattern auch benannt sein.
*Fix:* `outputs = { self, nixpkgs, home-manager, ... }:` – den fehlenden Namen ergänzen.

## Übung

1. Migriere die Channel-Konfiguration deiner Buch-VM nach dem Migrationspfad oben zu Flakes. Prüfe mit `nixos-rebuild switch --flake`, dass sich an der laufenden Konfiguration nichts ändert.
2. Aktualisiere gezielt nur den `nixpkgs`-Input mit `nix flake update nixpkgs`, nicht alle Inputs, und schau dir per `git diff flake.lock` an, was sich geändert hat.

**Lösungsskizze:**

Zu 1 und 2: Folge den Abschnitten "Migrationspfad" und "`nix flake update` im Detail" oben Schritt für Schritt.

## Zusammenfassung

- Channels sind ein rollender, nicht versionierter Zeiger – praktisch, aber ohne eingebaute exakte Reproduzierbarkeit.
- `flake.nix` beschreibt Inputs (woher kommt was) und Outputs (u. a. `nixosConfigurations`); `outputs` ist schlicht eine Funktion.
- `flake.lock` fixiert jeden Input auf einen exakten Commit- und Inhalts-Hash und gehört mit ins Git-Repo.
- `nix flake update` aktualisiert alle Inputs; ein Inputname als Argument aktualisiert gezielt nur diesen einen.
- Migration von Channels zu Flakes: `flake.nix` schreiben, die alte `configuration.nix` unverändert importieren, `--flake` nutzen – richtig gemacht ein No-op.
- Auch ohne Flakes lässt sich Nixpkgs über `builtins.fetchTarball` mit fester URL und Hash pinnen.
- Flakes sind trotz De-facto-Standard-Status formal weiterhin experimentell.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `nix-channel --add/--update`, Channel-Typen | [NixOS Manual – Upgrading NixOS](https://nixos.org/manual/nixos/stable/) |
| `flake.nix`-Struktur, `nixpkgs.lib.nixosSystem`, `follows` | [NixOS-Wiki – Flakes](https://wiki.nixos.org/wiki/Flakes), [NixOS & Flakes Book](https://nixos-and-flakes.thiscute.world/) |
| `flake.lock`-Aufbau (`rev`, `narHash`, `locked`/`original`) | [nixos.asia – Convert configuration.nix to be a flake](https://nixos.asia/en/configuration-as-flake) |
| `nix flake update`, `nix flake lock --update-input` | Nix Reference Manual; Praxisbeispiel (chenlijun99/dotfiles) |
| Migrationspfad Channels → Flakes | [nixos.asia – Convert configuration.nix to be a flake](https://nixos.asia/en/configuration-as-flake) |
| `builtins.fetchTarball`-Pinning | Allgemein dokumentiertes Nix-Muster |
| `nix flake init -t` | Nix Reference Manual |
