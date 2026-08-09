---
title: "Secrets"
weight: 10
---

# Secrets

## Lernziele

- Du kannst erklären, warum der Nix Store world-readable ist und was das für Geheimnisse bedeutet.
- Du kannst sops-nix aufsetzen und ein Secret in eine Konfiguration einbinden.
- Du kannst agenix einordnen und die Unterschiede zu sops-nix benennen.
- Du verwendest `hashedPasswordFile` statt eines Klartext- oder Store-Passworts.
- Du kennst den klassischen Fehler, der ein Secret trotz Tooling doch im Store landen lässt.

## Warum das wichtig ist

Kapitel 6 hat bewusst ausgeklammert, was ein *echtes* Geheimnis ist. Hier wird das nachgeholt – mit Werkzeugen, die zum NixOS-Modell passen (deklarativ, versionierbar, Teil desselben Rebuild-Workflows), statt Secrets manuell und undokumentiert an der Konfiguration vorbeizuschmuggeln.

## Warum der Store world-readable ist

Kurze Erinnerung an Kapitel 2: Store-Pfade sind für jeden lokalen Nutzer *lesbar*, nur nicht schreibbar. Das ist eine bewusste Design-Entscheidung, keine Nachlässigkeit – Builds müssen für jeden Nutzer nachvollziehbar und überprüfbar sein, und der Nix-Daemon baut im Auftrag verschiedener Nutzer gleichzeitig. Die Konsequenz: Alles, was direkt in `configuration.nix` landet – und damit über den Build-Prozess im Store – ist für jeden lokalen Account einsehbar. Ein API-Token oder ein Klartext-Passwort dort hineinzuschreiben, ist also nicht "ein bisschen unsauber", sondern schlicht keine Geheimhaltung. Genau diese Lücke schließen die Tools in diesem Kapitel: Sie halten Geheimnisse verschlüsselt im Repository und entschlüsseln sie erst bei der Aktivierung, an Orten außerhalb des Stores.

## sops-nix

[sops-nix](https://github.com/Mic92/sops-nix) baut auf Mozillas `sops` (Secrets OPerationS) auf: Eine Datei mit Geheimnissen wird verschlüsselt als YAML/JSON/INI/dotenv im Repository abgelegt, und ein NixOS-Modul entschlüsselt sie bei der Aktivierung.

**Einbinden als Flake-Input:**

```nix
inputs.sops-nix.url = "github:Mic92/sops-nix";
inputs.sops-nix.inputs.nixpkgs.follows = "nixpkgs";
```

Modul einbinden (in `configuration.nix` oder direkt in der `modules`-Liste):

```nix
imports = [ inputs.sops-nix.nixosModules.sops ];

sops = {
  defaultSopsFile = ./secrets/secrets.yaml;
  defaultSopsFormat = "yaml";
  age.keyFile = "/root/.config/sops/age/keys.txt";

  secrets.datenbank_passwort = {
    owner = "postgres";
  };
};
```

Bearbeitet wird die verschlüsselte Datei mit dem `sops`-Kommando selbst, das sie entschlüsselt, deinen `$EDITOR` öffnet und beim Speichern wieder verschlüsselt:

```console
$ sops secrets/secrets.yaml
```

Zur Laufzeit liegt das entschlüsselte Secret unter `/run/secrets/datenbank_passwort` – referenzierbar aus jeder anderen Option über `config.sops.secrets.datenbank_passwort.path`, nie als Klartext direkt in der Konfiguration.

## agenix als Alternative

[agenix](https://github.com/ryantm/agenix) verfolgt dieselbe Grundidee, aber bewusst schlanker: Es unterstützt ausschließlich `age` als Verschlüsselungs-Backend (kein GPG) und verzichtet auf Templating-Funktionen, um die Codebasis klein und auditierbar zu halten.

Eine `secrets.nix` im Repo-Root legt fest, welche (SSH- oder age-)Public-Keys welches `.age`-File entschlüsseln dürfen – diese Datei wird selbst *nicht* in die NixOS-Konfiguration importiert, sondern nur vom `agenix`-CLI zum Verschlüsseln benutzt:

```nix
# secrets.nix
let
  meinLaptop = "ssh-ed25519 AAAA...";
  meinServer = "ssh-ed25519 AAAA...";
in
{
  "datenbank-passwort.age".publicKeys = [ meinLaptop meinServer ];
}
```

In der NixOS-Konfiguration:

```nix
age.secrets.datenbank-passwort = {
  file = ./secrets/datenbank-passwort.age;
  owner = "postgres";
};

services.postgresql.initialScript = config.age.secrets.datenbank-passwort.path;
```

Entschlüsselt landet das Secret unter `/run/agenix/datenbank-passwort`.

**Der praktische Unterschied zu sops-nix:** agenix nutzt meist direkt vorhandene SSH-Host-Keys (Ed25519-SSH-Keys sind mit age kompatibel) – kein separates Schlüsselmanagement nötig, wenn du ohnehin SSH-Keys verteilst. sops-nix ist dafür flexibler (mehrere Dateiformate, GPG *und* age, laut Projekt auch Rollback-Unterstützung für im Store abgelegte sops-Dateien) und in größeren, heterogenen Setups oft die praktischere Wahl.

## `hashedPasswordFile`

Aus Kapitel 6 kennst du `users.users.<name>.hashedPassword` – der Hash landet dabei direkt in `configuration.nix` und damit im Store. Ein Hash ist kein Klartext, aber je nach Hash-Verfahren und Passwortstärke unter Umständen doch angreifbar, und landet obendrein für jeden lokal einsehbar im Store. `hashedPasswordFile` verweist stattdessen auf einen *Pfad* – ideal kombiniert mit sops-nix oder agenix:

```nix
users.users.alex.hashedPasswordFile = config.sops.secrets.alex-passwort.path;
```

Der Hash selbst existiert dann nur entschlüsselt zur Laufzeit unter `/run/secrets/…`, nie im Store.

## Vollständiges Beispiel

sops-nix komplett durchgespielt, von der Verschlüsselung bis zur Nutzung für ein Nutzerpasswort:

```console
$ mkdir -p ~/.config/sops/age
$ nix shell nixpkgs#age -c age-keygen -o ~/.config/sops/age/keys.txt
Public key: age1qyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqsyzhk3l
```

```yaml
# .sops.yaml im Repo-Root
creation_rules:
  - path_regex: secrets/.*\.yaml$
    key_groups:
      - age:
          - age1qyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqsyzhk3l
```

```console
$ sops secrets/secrets.yaml
```

Im Editor:

```yaml
alex-passwort: $6$rounds=..../hashedwert...
```

In `configuration.nix`:

```nix
{
  imports = [ inputs.sops-nix.nixosModules.sops ];

  sops = {
    defaultSopsFile = ./secrets/secrets.yaml;
    age.keyFile = "/root/.config/sops/age/keys.txt";
    secrets.alex-passwort = { };
  };

  users.users.alex.hashedPasswordFile = config.sops.secrets.alex-passwort.path;
}
```

Nach `nixos-rebuild switch` ist das Passwort aktiv – zu keinem Zeitpunkt stand der Hash unverschlüsselt in einer Datei, die in den Store gebaut wurde.

> ⚠️ **Der Fehler, der keine Fehlermeldung wirft:** `builtins.readFile config.sops.secrets.X.path` (oder das agenix-Äquivalent) *innerhalb* der Nix-Auswertung zu verwenden, sieht harmlos aus, kopiert das entschlüsselte Secret aber direkt in den – world-readable! – Store, weil `builtins.readFile` zur Build-Zeit läuft, nicht zur Aktivierungszeit. Es gibt dafür keine Fehlermeldung, keine Warnung – das System baut anstandslos, und genau das macht diesen Fehler gefährlich. Der Pfad (`config.sops.secrets.X.path` bzw. `config.age.secrets.X.path`) ist als *Laufzeit*-Pfad gedacht, den ein Dienst selbst einliest (z. B. über `PasswordFile =`-artige Optionen), nicht als etwas, das die Nix-Auswertung selbst öffnet.

> 💡 **Nice to know:** sops-nix unterstützt laut eigener Dokumentation sowohl **age** als auch **GPG** als Verschlüsselungs-Backend; agenix bewusst nur age. Für neue Projekte ist age fast immer die einfachere Wahl (kürzere Keys, kein Web-of-Trust-Overhead) – GPG lohnt sich vor allem, wenn du ohnehin schon GPG-Infrastruktur (z. B. Hardware-Tokens) im Einsatz hast.

> 💡 **Nice to know:** Dieselbe Mechanik – Secrets verschlüsselt im Repo, entschlüsselt erst zur Laufzeit – ist auch die Grundlage dafür, Secrets sicher in CI/CD-Pipelines einzusetzen, etwa wenn Kapitel 14 zeigt, wie `colmena` oder `deploy-rs` mehrere Maschinen automatisiert bespielen.

## Typische Fehler

**1. Falscher oder fehlender age-Schlüssel bei sops-nix:**

```
Failed to get the data key required to decrypt the SOPS file.
```

*Ursache:* `age.keyFile` zeigt auf keine oder eine falsche Datei, oder der zugehörige Public Key wurde nie als Empfänger in `.sops.yaml` hinterlegt.
*Fix:* Pfad zu `age.keyFile` prüfen; die Secrets-Datei ggf. mit dem korrekten Public Key neu verschlüsseln (`sops updatekeys`).

**2. Zielmaschine fehlt als Empfänger bei agenix:**

```
age: error: no identity matched any of the recipients
```

*Ursache:* Der SSH-Host-Key der Maschine wurde nicht (oder mit falschem Public Key) in `secrets.nix` als Empfänger eingetragen.
*Fix:* Aktuellen Public Key der Maschine (`/etc/ssh/ssh_host_ed25519_key.pub`) in `secrets.nix` ergänzen und das Secret neu verschlüsseln.

## Übung

1. Richte sops-nix in deiner Buch-VM ein: age-Schlüssel erzeugen, `secrets.yaml` anlegen, ein Test-Secret verschlüsseln, und binde es für einen beliebigen Dienst ein.
2. Ersetze – falls noch vorhanden – das Klartext- oder `hashedPassword`-Passwort deines Nutzers durch `hashedPasswordFile`, das auf das sops-verwaltete Secret zeigt.

**Lösungsskizze:**

Zu 1 und 2: Folge dem Abschnitt "Vollständiges Beispiel" oben Schritt für Schritt.

## Zusammenfassung

- Der Nix Store ist absichtlich world-readable – echte Geheimnisse dürfen deshalb nie direkt in `configuration.nix` landen.
- sops-nix verschlüsselt Secrets-Dateien (YAML/JSON/INI/dotenv) mit GPG oder age; entschlüsselt liegen sie unter `/run/secrets/…`.
- agenix ist die schlankere Alternative, ausschließlich mit age, oft direkt mit vorhandenen SSH-Host-Keys nutzbar; entschlüsselt liegen Secrets unter `/run/agenix/…`.
- `hashedPasswordFile` hält auch Passwort-Hashes aus dem Store heraus, kombiniert mit sops-nix oder agenix.
- Der gefährlichste Fehler ist unsichtbar: `builtins.readFile` auf einen Secret-Pfad kopiert den Klartext ohne jede Fehlermeldung in den Store.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `sops.*`, `sops.secrets.<name>`, `/run/secrets/…` | [sops-nix – GitHub](https://github.com/Mic92/sops-nix/), [Praxisbeispiel (Stapelberg)](https://michael.stapelberg.ch/posts/2025-08-24-secret-management-with-sops-nix/) |
| sops unterstützt GPG und age | [sops-nix README](https://github.com/Mic92/sops-nix/) |
| `age.secrets.<name>`, `secrets.nix`, `/run/agenix/…` | [NixOS-Wiki – Agenix](https://wiki.nixos.org/wiki/Agenix) |
| agenix nur age, kein Templating | [NixOS-Wiki – Agenix](https://wiki.nixos.org/wiki/Agenix) |
| Warnung vor `builtins.readFile` auf Secret-Pfaden | [woile.eu – NixOS with agenix](https://woile.eu/blog/agenix.html) |
| `users.users.<name>.hashedPasswordFile` | NixOS-Option, siehe search.nixos.org |
| `age: error: no identity matched any of the recipients` | Standard-`age`-Fehlermeldung (age-Projekt) |
