---
title: "Desktop (kompakt)"
weight: 15
---

# Desktop (kompakt)

## Lernziele

- Du kannst einen Display-Manager und ein Desktop-Environment mit den *aktuellen* Optionsnamen aktivieren.
- Du kennst den NVIDIA-Sonderfall grob.
- Du weißt, dass dieses Kapitel bewusst kein vollständiger Desktop-Guide ist.

## Warum das wichtig ist

Der Rest dieses Buchs ist Server/Headless-fokussiert – aber NixOS auf dem eigenen Laptop ist für viele der erste Kontaktpunkt überhaupt. Genau in diesem Bereich haben sich die Optionsnamen kürzlich stark verschoben, und veraltete Blogposts/Tutorials sind hier besonders häufig – ein kurzer, aktueller Überblick lohnt sich deshalb trotzdem.

## Display-Manager & Desktop-Environment: aktuelle Optionen

> ⚠️ Seit NixOS 25.11 liegen Display-Manager- und Desktop-Manager-Optionen auf **oberster Ebene** – `services.displayManager.*` und `services.desktopManager.*` – nicht mehr unter `services.xserver.*`. Sehr viele Tutorials im Netz (auch aktuell wirkende) zeigen noch die alte, verschachtelte Form. Beides direkt aus dem Nixpkgs-Quellcode des 26.05-Branches bestätigt.

GNOME:

```nix
services.displayManager.gdm.enable = true;
services.desktopManager.gnome.enable = true;
```

KDE Plasma 6:

```nix
services.displayManager.sddm.enable = true;
services.displayManager.sddm.wayland.enable = true;
services.desktopManager.plasma6.enable = true;
```

Für beide gilt: `services.xserver.enable = true;` wird oft trotzdem noch gebraucht – nicht für den Display-Manager selbst, sondern für Xwayland (X11-Kompatibilität unter Wayland) und als Fallback für rein X11-basierte Anwendungen.

## Tastaturlayout für die grafische Oberfläche

Das aus Kapitel 6 versprochene Gegenstück zu `console.keyMap` (das nur die Textkonsole betrifft):

```nix
services.xserver.xkb.layout = "de";
services.xserver.xkb.variant = "";   # z. B. "nodeadkeys", optional
```

Anders als die Display-/Desktop-Manager-Optionen ist diese Option (Stand 26.05) unter `services.xserver` verblieben, da sie an das X-Server-Protokoll gebunden ist – auch unter Wayland läuft die Tastaturbelegung über dieselbe xkb-Konfiguration.

## Grafiktreiber & der NVIDIA-Sonderfall

```nix
services.xserver.videoDrivers = [ "nvidia" ];
hardware.graphics.enable = true;   # bis vor Kurzem: hardware.opengl.enable

hardware.nvidia = {
  modesetting.enable = true;
  open = true;   # NVIDIA-eigene Open-Source-Kernel-Module statt proprietär
};
```

`hardware.graphics` (der Nachfolgename von `hardware.opengl`, das noch in älteren Anleitungen auftaucht) aktiviert den grafischen Unterbau – ohne diese Option scheitern OpenGL-/Vulkan-Anwendungen selbst mit korrekt geladenem Treiber. NVIDIA bleibt der Sonderfall unter den Grafiktreibern: proprietär, mit eigener Optionsgruppe (`hardware.nvidia.*`), und gerade im Zusammenspiel mit Wayland historisch die Quelle der meisten Desktop-bezogenen Bugreports im NixOS-Bugtracker.

## Bewusste Abgrenzung

Dieses Kapitel ist ein Einstiegspunkt, kein vollständiger Desktop-Guide – Themen wie Extensions, Theming, Multi-Monitor-Setups oder Audio-Feinkonfiguration (PipeWire) bleiben bewusst außen vor. Die [NixOS-Wiki-Kategorie "Desktop environment"](https://wiki.nixos.org/wiki/Category:Desktop_environment) sowie die jeweilige DE-spezifische Wiki-Seite (GNOME, KDE, …) sind die richtigen Anlaufstellen dafür.

> 💡 **Nice to know:** Sowohl GNOME als auch Plasma 6 laufen inzwischen standardmäßig auf **Wayland**, nicht mehr auf X11 – ein Wandel, der erst in den letzten NixOS-Releases wirklich abgeschlossen wurde. `services.xserver.enable` bleibt trotzdem relevant, primär für Xwayland.

> 💡 **Nice to know:** Für Laptops und Einplatinencomputer mit hardwarespezifischen Eigenheiten (Touchpad-Feintuning, Zwei-GPU-Setups, spezielle Firmware) lohnt sich – wie schon in Kapitel 4 erwähnt – ein Blick ins Repository [NixOS/nixos-hardware](https://github.com/NixOS/nixos-hardware), bevor du bei null anfängst.

## Typische Fehler

**1. Veraltete, verschachtelte Option aus einem alten Tutorial übernommen:**

```
The option `services.xserver.displayManager.gdm.enable' defined in `/etc/nixos/configuration.nix' does not exist.
```

*Ursache:* Seit 25.11 liegen diese Optionen auf oberster Ebene (siehe oben), die alte, verschachtelte Form wurde entfernt.
*Fix:* `services.displayManager.gdm.enable` bzw. `services.desktopManager.<name>.enable` verwenden.

**2. `hardware.graphics.enable` vergessen:**

```
libGL error: failed to load driver
```

*Ursache:* Ohne aktivierten Grafik-Stack fehlt OpenGL-/Vulkan-Anwendungen die nötige Treiberanbindung, selbst wenn der eigentliche Kernel-Treiber korrekt geladen ist.
*Fix:* `hardware.graphics.enable = true;` setzen.

## Übung

Aktiviere in einer separaten Test-VM (mit grafischer Ausgabe, nicht der headless Buch-VM) GNOME oder KDE Plasma 6 mit den aktuellen Optionen aus diesem Kapitel und rebuilde.

**Lösungsskizze:** Folge dem jeweiligen Codeblock oben unter "Display-Manager & Desktop-Environment".

## Zusammenfassung

- Seit 25.11 liegen Display-/Desktop-Manager-Optionen auf oberster Ebene (`services.displayManager.*`, `services.desktopManager.*`), nicht mehr unter `services.xserver`.
- `services.xserver.enable` bleibt meist trotzdem nötig, primär für Xwayland.
- `hardware.graphics.enable` (Nachfolger von `hardware.opengl.enable`) aktiviert den Grafik-Unterbau; NVIDIA bleibt mit eigener Optionsgruppe der Sonderfall.
- GNOME und Plasma 6 laufen inzwischen standardmäßig auf Wayland.
- Dies ist ein Einstiegspunkt, kein vollständiger Desktop-Guide – NixOS-Wiki und `nixos-hardware` sind die Anlaufstellen für alles Weitere.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `services.displayManager.gdm.enable`, `services.desktopManager.gnome.enable` | [Nixpkgs-Quellcode, release-26.05](https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/desktop-managers/gnome.nix), [NixOS-Wiki – Desktop environment](https://wiki.nixos.org/wiki/Category:Desktop_environment) |
| `services.desktopManager.plasma6.enable`, `services.displayManager.sddm.wayland.enable` | [Nixpkgs-Quellcode, release-26.05](https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/desktop-managers/plasma6.nix), [MyNixOS](https://mynixos.com/nixpkgs/option/services.desktopManager.plasma6.enable), [NixOS-Wiki – KDE](https://wiki.nixos.org/wiki/KDE) |
| `hardware.graphics.enable` (vormals `hardware.opengl.enable`) | [Nixpkgs-Issue #323396](https://github.com/NixOS/nixpkgs/issues/323396) |
| `hardware.nvidia.*`, `services.xserver.videoDrivers` | [Nixpkgs-Issues #323396, #295218](https://github.com/NixOS/nixpkgs/issues/295218) |
| `NixOS/nixos-hardware` | https://github.com/NixOS/nixos-hardware |
