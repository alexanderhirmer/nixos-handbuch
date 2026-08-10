---
title: "Lokaler Admin-Nutzer"
weight: 3
---

# Schritt 3: Der lokale Admin existiert, ausschließlich mit SSH-Key

## Ziel

Der lokale Nutzer `<admin-user>` (Muster `f-local-admin-<hostname>`) ist deklarativ angelegt, hat einen hinterlegten SSH-Key und keinerlei funktionierendes Passwort – weder gesetzt noch erratbar.

## Voraussetzung

Schritt 2 ist abgeschlossen: `nixos-rebuild switch --flake /etc/nixos#<hostname>` lief bereits erfolgreich, der Container hat die feste IP `<ip>` und `pkgs.git` ist installiert.

## Durchführung

**1. Ein SSH-Schlüsselpaar für `<admin-user>` bereitstellen** (auf dem Rechner, von dem später administriert wird – falls noch nicht vorhanden):

```console
$ ssh-keygen -t ed25519 -C "<admin-user>"
```

Der öffentliche Schlüssel (`~/.ssh/id_ed25519.pub` oder wo auch immer abgelegt) wandert unten in `users.nix`.

**2. Modul-Dateien lokal schreiben** (siehe "Dateien").

**3. Dateien in den Container bringen und aktivieren** – ab jetzt ist `git` im Container verfügbar (Schritt 2), trotzdem bleibt `pct push` das etablierte, bereits geprüfte Mittel für diesen Schritt:

```console
$ pct exec <vmid> -- mkdir -p /etc/nixos/modules/baseline
$ pct push <vmid> <repo-root>/modules/baseline/default.nix \
    /etc/nixos/modules/baseline/default.nix
$ pct push <vmid> <repo-root>/modules/baseline/users.nix \
    /etc/nixos/modules/baseline/users.nix
$ pct push <vmid> <repo-root>/hosts/<hostname>/configuration.nix \
    /etc/nixos/hosts/<hostname>/configuration.nix
$ pct enter <vmid>
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

## Dateien

```nix
# <repo-root>/modules/baseline/default.nix
{
  imports = [
    ./users.nix
  ];
}
```

```nix
# <repo-root>/modules/baseline/users.nix
{ ... }:
{
  # Ab hier ist jede Kontoänderung ausschließlich über diese Datei erlaubt –
  # kein `useradd`, kein `passwd` auf der laufenden Maschine.
  users.mutableUsers = false;

  users.users."<admin-user>" = {
    isNormalUser = true;
    description = "Lokaler Fallback-Admin (kein LDAP)";
    extraGroups = [ "wheel" ]; # nötig für die Lockout-Schutzregel, s. u. – NICHT gleichbedeutend mit Sudo-Rechten
    hashedPassword = null; # siehe Erklärung unten
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... <admin-user>"
    ];
  };
}
```

```diff
--- a/hosts/<hostname>/configuration.nix
+++ b/hosts/<hostname>/configuration.nix
@@
   imports = [
     (modulesPath + "/virtualisation/proxmox-lxc.nix")
+    ../../modules/baseline
   ];
```

**Zur `hashedPassword = null` vs. `"!"`-Frage:** `hashedPassword = null` ist der dokumentierte Default und bedeutet laut Options-Beschreibung im Nixpkgs-Quellcode: "this user will not be able to log in using a password". Was in `/etc/shadow` landet, entscheidet aber `users.mutableUsers`, nicht `hashedPassword` allein – im aktivierenden Perl-Skript `update-users-groups.pl` steht wörtlich: `$sp_pwdp = "!" if !$spec->{mutableUsers};`, nur überschrieben, falls `hashedPassword` tatsächlich gesetzt ist.<sup>1</sup> Mit `mutableUsers = false` und `hashedPassword = null` landet also so oder so ein literales `!` im Passwortfeld – der Standard-Shadow-Marker für "kein gültiger Hash, Passwort-Login unmöglich" (zu unterscheiden von einem leeren Feld `""`, das laut derselben Quelle *passwortloses* Login erlauben würde). Explizit `hashedPassword = "!";` zu schreiben wäre gleichwertig; `null` ist hier vorzuziehen, weil es die dokumentierte, für diesen Zweck vorgesehene Standardeinstellung ist.

Wichtig: Das sperrt nur *dieses eine Konto*. `services.openssh.settings.PasswordAuthentication` bleibt systemweit aktiv, weil LDAP-Nutzer sich später (Schritt 8) per Passwort anmelden sollen – die Sperre für `<admin-user>` wirkt rein über den fehlenden gültigen Hash in `/etc/shadow`, nicht über eine globale SSH-Einstellung.

> 💡 **Nice to know:** `extraGroups = [ "wheel" ]` ist hier *nicht* die Quelle des Sudo-Rechts – das kommt in Schritt 4 gezielt über `security.sudo.extraRules`. Ohne mindestens ein Konto in `wheel` (oder `root`) mit Passwort/Key würde der Build bei `mutableUsers = false` mit einer Assertion abbrechen (Lockout-Schutz, siehe unten) – deshalb steht die Gruppenmitgliedschaft schon hier.

## Prüfen

`pct exec <vmid> -- id <admin-user>` zeigt eine UID ≥ 1000 und `wheel` in der Gruppenliste. `pct exec <vmid> -- getent shadow <admin-user>` zeigt im zweiten, doppelpunktgetrennten Feld exakt `!`. Ein Passwort-Login ist damit unmöglich; ein echter SSH-Verbindungstest von außen ist an dieser Stelle noch nicht aussagekräftig, weil Port und Firewall erst in den Schritten 5 und 9 stehen.

## Wenn's schiefgeht

**Build bricht ab mit `Neither the root account nor any wheel user has a password or SSH authorized key. You must set one to prevent being locked out of your system.`**: `extraGroups = [ "wheel" ]` fehlt, oder die `authorizedKeys.keys`-Liste ist leer – NixOS verweigert bei `mutableUsers = false` einen Zustand, der garantiert aussperrt. Fix: Gruppe bzw. Key ergänzen.

**`users.mutableUsers = false;` vergessen:** Kein Build-Fehler, aber die Sicherheitsgarantie gilt nicht mehr zuverlässig – ein späteres, manuelles `passwd <admin-user>` auf der laufenden Maschine würde nicht beim nächsten Rebuild zurückgesetzt (klassischer Config-Drift, Kapitel 1). Fix: Zeile ergänzen, rebuilden.

**Publickey-Login schlägt später fehl, obwohl der Key korrekt aussieht:** Meist ein fehlender Typ-Präfix (`ssh-ed25519 …`) oder ein Zeilenumbruch mitten im Key beim Kopieren. Der Build selbst meldet dabei nichts – `authorizedKeys.keys` landet als Text ohne Formatprüfung, `sshd` ignoriert eine kaputte Zeile erst beim Verbindungsversuch stillschweigend.

## Rückweg

Den `users.users."<admin-user>"`-Block aus `users.nix` entfernen (oder den ganzen `../../modules/baseline`-Import aus `configuration.nix`), erneut pushen und rebuilden. `update-users-groups.pl` entfernt den Eintrag danach aus `/etc/passwd`, `/etc/group` und `/etc/shadow`; das Home-Verzeichnis unter `/home/<admin-user>` bleibt davon unberührt und muss bei Bedarf separat gelöscht werden.

## Querverweis

Deklarative Nutzerverwaltung, `isNormalUser`, `openssh.authorizedKeys.keys`, `mutableUsers`: Teil I, Kapitel 6 ("Alltagsbetrieb"). `imports` und Modul-Zusammenführung: Kapitel 5.

---

<sup>1</sup> Quelle: Nixpkgs-Quellcode, `nixos/modules/config/users-groups.nix` (Options-Beschreibung von `hashedPassword`, `allowsLogin`-Funktion, Lockout-Assertion) und `nixos/modules/config/update-users-groups.pl` (Zeilen zur `/etc/shadow`-Erzeugung), beide Branch `release-26.05`, verbatim geladen über `raw.githubusercontent.com`.
