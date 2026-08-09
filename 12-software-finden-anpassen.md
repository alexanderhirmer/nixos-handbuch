---
title: "Software finden & anpassen"
weight: 12
---

# Software finden & anpassen

## Lernziele

- Du nutzt search.nixos.org gezielt für Pakete *und* Optionen.
- Du verstehst, was ein Overlay ist, und kannst ein eigenes schreiben.
- Du kannst `override` von `overrideAttrs` unterscheiden und richtig einsetzen.
- Du schaltest unfreie und unsichere Pakete gezielt frei.
- Du baust ein eigenes Minimalpaket mit `stdenv.mkDerivation`.

## Warum das wichtig ist

Nicht jedes Paket, das du brauchst, ist bereits fertig in Nixpkgs, exakt in der gewünschten Version, oder überhaupt frei lizenziert. Dieses Kapitel schließt die Lücke zwischen "Paket existiert und passt" (Kapitel 6) und "Paket existiert nicht oder passt nicht ganz".

## search.nixos.org effektiv nutzen

Zwei getrennte Bereiche, leicht zu verwechseln: `search.nixos.org/packages` für Pakete, `search.nixos.org/options` für NixOS-Optionen. Wer nach einer Konfigurationsoption im Paket-Tab sucht, findet zuverlässig nichts Passendes. Beide Suchen lassen sich außerdem nach Channel filtern (`unstable` vs. `26.05`) – die Ergebnisse unterscheiden sich, gerade bei neueren Paketen, die es in die Stable-Version noch nicht geschafft haben.

## Overlays: Konzept & eigenes Overlay schreiben

Ein **Overlay** ist eine Funktion der Form `final: prev: { … }`, die das *gemeinsame* `pkgs`-Set verändert – im Gegensatz zu `override`/`overrideAttrs` (siehe unten), die nur eine lokale Kopie für eine einzelne Verwendungsstelle erzeugen. Ändert ein anderes Paket in deiner Konfiguration selbst etwas über `pkgs.foo`, sieht es mit einem Overlay automatisch deine angepasste Version; ohne Overlay bekäme es weiterhin das Original.

```nix
# In flake.nix oder als eigenständiges overlay.nix importiert
final: prev: {
  hello = prev.hello.overrideAttrs (finalAttrs: previousAttrs: {
    pname = previousAttrs.pname + "-custom";
  });
}
```

Eingebunden über die NixOS-Option:

```nix
nixpkgs.overlays = [ (import ./overlay.nix) ];
```

`final` ist das *fertige*, alle Overlays bereits berücksichtigende Set (nützlich, wenn dein Overlay auf einem anderen Overlay aufbauen soll), `prev` das Set *vor* diesem einen Overlay – für die meisten einfachen Anpassungen brauchst du nur `prev`.

## `overrideAttrs` vs. `override`

Die beiden werden ständig verwechselt, tun aber unterschiedliche Dinge:

- **`override { … }`** ändert die *Argumente*, mit denen die paketerzeugende Funktion aufgerufen wurde – z. B. ein Feature-Flag, das ein Paket explizit als Parameter anbietet: `pkgs.foo.override { barSupport = true; }`.
- **`overrideAttrs (finalAttrs: previousAttrs: { … })`** ändert direkt das Attribut-Set, das an `stdenv.mkDerivation` übergeben wird – Version, Quelltext, Patches, Build-Inputs, praktisch alles.

```nix
# override: ein exponiertes Funktionsargument ändern
mySed = pkgs.gnused.override { /* z. B. ein Build-Flag */ };

# overrideAttrs: die Derivation selbst anpassen
helloWithDebug = pkgs.hello.overrideAttrs (finalAttrs: previousAttrs: {
  separateDebugInfo = true;
});
```

Eine dritte, ältere Funktion, `overrideDerivation`, taucht in älteren Tutorials noch auf – sie gilt nicht als deprecated, aber das Nixpkgs Manual selbst empfiehlt inzwischen durchgängig `overrideAttrs` statt ihrer. Begegnet sie dir in einer Anleitung, ersetze sie im Kopf gedanklich durch `overrideAttrs`.

> 💡 Praktischer Kniff aus Kapitel 11: `nix repl -f '<nixpkgs>'`, dann `:e stdenv.mkDerivation` (oder `:e pkgs.hello`) öffnet den tatsächlichen Nixpkgs-Quellcode in deinem Editor – oft schneller als raten, welche Attribute überhaupt existieren.

## Unfreie und unsichere Pakete freischalten

Nixpkgs verweigert standardmäßig die Auswertung unfreier und als unsicher markierter Pakete – nicht aus technischen Gründen, sondern als bewusste Warnung. Freischalten:

```nix
nixpkgs.config.allowUnfree = true;
nixpkgs.config.permittedInsecurePackages = [ "openssl-1.0.2u" ];
```

`permittedInsecurePackages` braucht den *exakten* Paketnamen samt Version, nicht nur den Basisnamen – den findest du in der Fehlermeldung selbst (siehe "Typische Fehler" unten).

## Ein eigenes Minimalpaket bauen

```nix
pkgs.stdenv.mkDerivation {
  pname = "buch-gruss";
  version = "1.0";

  src = ./buch-gruss;   # Verzeichnis mit einer Datei "buch-gruss.sh"

  installPhase = ''
    mkdir -p $out/bin
    install -m755 buch-gruss.sh $out/bin/buch-gruss
  '';
}
```

`$out` ist der Store-Pfad, den Nix für diese Derivation reserviert (Kapitel 2) – alles, was ein anderes Paket später von diesem Paket braucht, muss unter `$out` landen, sonst existiert es aus Sicht von Nix schlicht nicht.

## Vollständiges Beispiel

Eigenes Paket, eingebunden über ein Overlay, systemweit installiert:

```nix
# overlay.nix
final: prev: {
  buch-gruss = prev.stdenv.mkDerivation {
    pname = "buch-gruss";
    version = "1.0";
    src = ./buch-gruss;
    installPhase = ''
      mkdir -p $out/bin
      install -m755 buch-gruss.sh $out/bin/buch-gruss
    '';
  };
}
```

```nix
# configuration.nix
{
  nixpkgs.overlays = [ (import ./overlay.nix) ];
  environment.systemPackages = [ pkgs.buch-gruss ];
}
```

Nach `nixos-rebuild switch` ist `buch-gruss` als ganz normaler Befehl verfügbar – nicht anders, als wäre es Teil von Nixpkgs selbst.

> 💡 **Nice to know:** `nixpkgs-review` baut genau die Pakete, die eine lokale Änderung (z. B. eine offene Nixpkgs-PR) tatsächlich betrifft, statt gleich den ganzen Baum neu zu bauen – der Standard-Workflow für alle, die selbst zu Nixpkgs beitragen.

> 💡 **Nice to know:** `pkgs.buildFHSEnv` (in älteren Anleitungen noch `buildFHSUserEnv`) baut eine Umgebung mit klassischem FHS-Layout (`/usr/lib`, `/lib`, …) für Binaries, die nicht damit klarkommen, dass es diese Pfade auf NixOS so nicht gibt – genau das Problem aus Kapitel 1 ("Software mit nicht-FHS-konformem dynamischem Linking").

## Typische Fehler

**1. Unfreies Paket ohne Freischaltung:**

```
error: Package 'X' has an unfree license ('unfree'), refusing to evaluate.
```

*Ursache:* `nixpkgs.config.allowUnfree` ist nicht gesetzt.
*Fix:* `nixpkgs.config.allowUnfree = true;` (oder gezielter mit `allowUnfreePredicate` nur für einzelne Pakete).

**2. Als unsicher markiertes Paket ohne Freischaltung:**

```
error: Package 'openssl-1.0.2u' is marked as insecure, refusing to evaluate.
```

*Ursache:* Bekannte Sicherheitslücke, Nixpkgs verweigert standardmäßig genau wie bei unfreien Paketen.
*Fix:* Den exakten Namen aus der Fehlermeldung in `nixpkgs.config.permittedInsecurePackages` eintragen – und kurz überlegen, ob das wirklich nötig ist.

## Übung

1. Suche auf search.nixos.org nach einem Paket, das du kennst, und folge dem Link zum Nixpkgs-Quellcode – such dort nach `pname`, `version` und `src`.
2. Übernimm das `buch-gruss`-Beispiel oben (oder ein eigenes Minimalpaket) in deine Buch-VM und mach es systemweit verfügbar.

**Lösungsskizze:**

Zu 2: Folge dem Abschnitt "Vollständiges Beispiel" oben.

## Zusammenfassung

- search.nixos.org trennt Pakete und Optionen in zwei Bereiche – und filtert nach Channel.
- Overlays (`final: prev: { … }`) ändern das gemeinsame `pkgs`-Set global; `override`/`overrideAttrs` erzeugen nur eine lokale Kopie.
- `override` ändert Funktionsargumente, `overrideAttrs` die tatsächlichen `mkDerivation`-Attribute – `overrideDerivation` gilt als veraltet zugunsten von `overrideAttrs`.
- Unfreie und unsichere Pakete müssen explizit freigeschaltet werden, mit dem exakten Namen aus der jeweiligen Fehlermeldung.
- Ein eigenes Minimalpaket ist letztlich nur `stdenv.mkDerivation` mit einer `installPhase`, die Dateien nach `$out` kopiert.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| Overlays (`final: prev`), `nixpkgs.overlays` | [NixOS & Flakes Book – Overlays](https://nixos-and-flakes.thiscute.world/nixpkgs/overlays) |
| `override` vs. `overrideAttrs`, `overrideDerivation` | [Nixpkgs Manual – Overriding](https://nixos.org/nixpkgs/manual/) |
| `nixpkgs.config.allowUnfree`, `permittedInsecurePackages` | NixOS-Option, siehe search.nixos.org; Praxisbeispiel Kapitel 9 |
| `stdenv.mkDerivation`, `$out` | Nixpkgs Manual – Standard Environment |
| `nixpkgs-review` | Community-Tool, siehe Projekt-Repository |
| `buildFHSEnv`/`buildFHSUserEnv` | Nixpkgs-Quellcode/-Dokumentation |
