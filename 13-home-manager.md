---
title: "Home Manager"
weight: 13
---

# Home Manager

## Lernziele

- Du kannst wiederholen, was Home Manager ist – und was es bewusst nicht ist.
- Du unterscheidest Standalone- von NixOS-Modul-Integration und kannst beide einrichten.
- Du setzt ein einfaches Shell-Dotfiles-Beispiel um.
- Du kannst einschätzen, wann sich Home Manager auf einem Server überhaupt lohnt.
- Du behältst die Release-Zyklus-Kompatibilität zwischen Home Manager und NixOS im Blick.

## Warum das wichtig ist

Home Manager ist der natürliche nächste Schritt, wenn Kapitel 6's `users.users` plus SSH-Keys nicht reicht, weil du auch die *interaktive* Umgebung eines Nutzers – Shell, Editor, Git-Konfiguration – deklarativ und reproduzierbar haben willst. Aber es ist eben kein Teil von NixOS selbst, sondern ein eigenständiges Projekt mit eigenem Lebenszyklus – und genau das bringt eigene Fallstricke mit.

## Was Home Manager ist – und was nicht

Aus Kapitel 1 in Erinnerung gerufen: Home Manager wendet denselben deklarativen Nix-Ansatz auf *Nutzer*-Ebene an – Dotfiles, Shell-Konfiguration, persönliche Programme – statt auf Systemebene. Es ist kein Ersatz für `users.users` (Kapitel 6): Die Kontenverwaltung selbst (Nutzer anlegen, SSH-Keys, Gruppenzugehörigkeit) bleibt NixOS' Aufgabe. Home Manager kommt erst danach ins Spiel, für das, was ein Nutzer in seiner eigenen Sitzung sieht.

## Standalone vs. NixOS-Modul-Integration

Zwei grundverschiedene Wege, mit unterschiedlichen Trade-offs:

**Standalone** – Home Manager als eigenständiges Tool, unabhängig von `nixos-rebuild`:

```console
$ home-manager switch
```

Auf allem außer NixOS und nix-darwin (also jedem "normalen" Linux, das nur den Nix-Paketmanager hat) ist das der *einzige* verfügbare Weg. Auch auf NixOS selbst sinnvoll, wenn du deine Nutzerumgebung bewusst unabhängig vom System-Rebuild verwalten willst.

**NixOS-Modul** – direkt in `configuration.nix` bzw. `flake.nix` eingebunden:

```nix
home-manager.users.eve = { pkgs, ... }: {
  home.packages = [ pkgs.atool pkgs.httpie ];
  programs.bash.enable = true;
  home.stateVersion = "26.05";
};
```

Wird bei jedem `nixos-rebuild switch` automatisch mitgebaut – kein separater `home-manager switch`-Aufruf nötig. Der Vorteil: ein einziger Befehl für System *und* Nutzerumgebung, ein gemeinsames Rollback (Kapitel 7) für beides. Der Nachteil: Jede noch so kleine Dotfile-Änderung erzeugt eine neue System-Generation – wer viel an seiner Shell-Konfiguration feilt, sammelt so schnell sehr viele Generationen an.

## Beispiel: Shell-Dotfiles

```nix
home-manager.users.alex = { pkgs, ... }: {
  home.stateVersion = "26.05";

  programs.bash = {
    enable = true;
    shellAliases = {
      ll = "ls -la";
      gs = "git status";
    };
  };

  programs.git = {
    enable = true;
    userName = "Alex";
    userEmail = "alex@example.org";
  };

  home.packages = [ pkgs.htop pkgs.ripgrep ];
};
```

Nach einem Rebuild hat `alex` sofort seine gewohnten Aliase, eine konfigurierte Git-Identität und zwei zusätzliche Pakete – ganz ohne manuell irgendwo `.bashrc` oder `.gitconfig` zu editieren.

## Wann lohnt sich Home Manager auf einem Server?

Ehrlich abgewogen, gerade für den Server/Headless-Fokus dieses Buchs:

- **Lohnt sich**, wenn mehrere Admins sich denselben Server teilen und jeder seine eigene, gewohnte Shell-/Tool-Konfiguration reproduzierbar haben will – jeder bekommt einfach sein eigenes `home-manager.users.<name>`.
- **Lohnt sich**, wenn du dieselbe persönliche Konfiguration über mehrere Maschinen hinweg identisch halten willst, Server eingeschlossen – ein Repository, viele Zielsysteme.
- **Lohnt sich eher nicht** bei einem Single-Admin-Server mit rein administrativer, seltener interaktiver Nutzung – der zusätzliche Konzept- und Wartungsaufwand steht dort oft in keinem Verhältnis zum Nutzen. Ein einfaches Dotfiles-Repository oder gar nichts reicht dann meist.

## Vollständiges Beispiel

Vollständige Flake-Integration, System und Home Manager gemeinsam:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    home-manager = {
      url = "github:nix-community/home-manager/release-26.05";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.buch-vm = nixpkgs.lib.nixosSystem {
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager
        {
          home-manager.useGlobalPkgs = true;
          home-manager.useUserPackages = true;
          home-manager.users.alex = import ./home.nix;
        }
      ];
    };
  };
}
```

`useGlobalPkgs = true;` sorgt dafür, dass Home Manager dasselbe `pkgs`-Set wie das System verwendet, statt sein eigenes zu instanziieren – spart Store-Platz und vermeidet doppelt gebaute Pakete in leicht unterschiedlichen Versionen.

> 💡 **Nice to know:** Home Manager pflegt eigene Release-Branches, die – analog zu NixOS – den Nixpkgs-Branches entsprechen: `release-26.05` gehört zu `nixos-26.05`, `master` zu `nixos-unstable`. `inputs.home-manager.inputs.nixpkgs.follows = "nixpkgs";` (siehe oben) sorgt dafür, dass beide garantiert dieselbe Nixpkgs-Revision verwenden – ohne das riskierst du, zwei leicht unterschiedliche Nixpkgs-Stände gleichzeitig im Store zu haben.

## Typische Fehler

**1. `home.stateVersion` nicht gesetzt:**

```
Failed assertions:
- home.stateVersion ist nicht gesetzt. (sinngemäß – Wortlaut variiert je nach Home-Manager-Version)
```

*Ursache:* `home.stateVersion` hat bewusst keinen Default – analog zu NixOS' `system.stateVersion` markiert es, mit welcher Home-Manager-Version deine Konfiguration kompatibel ist.
*Fix:* Einmalig auf die zum Zeitpunkt der Ersteinrichtung aktuelle Version setzen (hier `"26.05"`) und laut offizieller Doku *nicht* nachträglich einfach hochsetzen, ohne vorher die Release Notes von Home Manager selbst zu prüfen.

**2. `home-manager.nixosModules.home-manager` nicht importiert, aber `home-manager.users.<name>` trotzdem gesetzt:**

```
The option `home-manager.users.eve' defined in `/etc/nixos/configuration.nix' does not exist.
```

*Ursache:* Ohne das Modul kennt NixOS die Option `home-manager.users` schlicht nicht – derselbe Fehlertyp wie in Kapitel 5, nur diesmal verursacht durch ein fehlendes externes Modul statt einen Tippfehler.
*Fix:* `home-manager.nixosModules.home-manager` zur `modules`-Liste hinzufügen (siehe "Vollständiges Beispiel" oben).

## Übung

1. Richte Home Manager als NixOS-Modul für deinen Nutzer in der Buch-VM ein, mit mindestens `programs.bash.enable` und einem zusätzlichen Paket über `home.packages`.
2. Überlege für einen (ggf. hypothetischen) Server-Anwendungsfall aus deinem Umfeld: Würde sich Home Manager dort lohnen, oder reicht der einfache Weg aus Kapitel 6? Begründe kurz anhand der Abwägung oben.

**Lösungsskizze:**

Zu 1: Folge dem Abschnitt "Vollständiges Beispiel" oben.

Zu 2: Individuell – es gibt hier keine pauschal richtige Antwort, nur eine begründete Abwägung.

## Zusammenfassung

- Home Manager ist ein eigenständiges Projekt für Nutzer-Ebene, kein Ersatz für `users.users`.
- Standalone (`home-manager switch`) läuft unabhängig vom System-Rebuild und ist auf Nicht-NixOS-Systemen die einzige Option; NixOS-Modul-Integration baut alles gemeinsam mit `nixos-rebuild switch`.
- `home.stateVersion` ist Pflicht, ohne Default, und sollte nach der Ersteinrichtung nicht gedankenlos verändert werden.
- Auf Servern lohnt sich Home Manager vor allem bei mehreren Admin-Nutzern oder wenn dieselbe Konfiguration über mehrere Maschinen reproduziert werden soll – nicht automatisch überall.
- `inputs.home-manager.inputs.nixpkgs.follows = "nixpkgs";` und `useGlobalPkgs = true;` halten System und Home Manager auf derselben Nixpkgs-Revision.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `home-manager.users.<name>`, `home.stateVersion`, Beispielkonfiguration | [Home Manager Manual – NixOS module](https://nix-community.github.io/home-manager/installation/nixos.html) |
| `home-manager switch` (Standalone), Installationswege im Überblick | [Home Manager Manual](https://home-manager.dev/manual/25.05/), [GitHub – nix-community/home-manager](https://github.com/nix-community/home-manager) |
| Flake-Integration, `useGlobalPkgs`, `useUserPackages` | [NixOS & Flakes Book – Getting Started with Home Manager](https://nixos-and-flakes.thiscute.world/nixos-with-flakes/start-using-home-manager) |
| Release-Branches (`release-26.05` ↔ `nixos-26.05`) | [NixOS-Wiki – Home Manager](https://wiki.nixos.org/wiki/Home_Manager) |
| `home.stateVersion` ohne Default (Pflichtoption) | [NixOS-Wiki – Home Manager](https://wiki.nixos.org/wiki/Home_Manager) |
