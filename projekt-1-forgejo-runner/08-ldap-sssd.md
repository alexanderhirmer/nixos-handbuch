---
title: "LDAP-Anbindung mit sssd"
weight: 8
---

# Schritt 8: sssd bindet den Container an das bestehende LDAP an

## Ziel

`sssd` bindet den Container an das bestehende LDAP an; ein LDAP-Testnutzer kann sich per Passwort per SSH einloggen, obwohl er nirgends in `users.users` auftaucht.

## Voraussetzung

Schritte 1–7 sind abgeschlossen: Das Flake baut, `<admin-user>` existiert mit SSH-Key und passwortlosem `sudo`, SSH läuft auf Port `<ssh-port>` mit deaktiviertem `PermitRootLogin` und aktivierter Passwort-Authentifizierung (nötig, sonst kommen LDAP-Nutzer gar nicht erst zur Passwortabfrage), `fail2ban` ist aktiv. Der bestehende LDAP-Server unter `<ldap-uri>` ist vom Container aus erreichbar und enthält mindestens einen Testnutzer mit vollständigen POSIX-Attributen (`uid`, `uidNumber`, `gidNumber`, `homeDirectory`) unterhalb von `<ldap-base-dn>`.

## Durchführung

**1. Bind-Modus klären.** Vom Container aus testen, ob ein anonymer Such-Bind reicht:

```console
$ ldapsearch -x -H <ldap-uri> -b <ldap-base-dn> -s base
```

Kommt ein Ergebnis ohne "Invalid credentials", genügt anonymes Lesen und `ldap.nix` braucht keinen Bind-DN. Verlangt der Server eine Authentifizierung, ist ein Bind-DN samt Passwort nötig (siehe Warnbox unten).

**2. `ldap.nix` anlegen** (Inhalt siehe Abschnitt „Dateien"), danach in `default.nix` importieren, testweise bauen und aktivieren:

```console
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

**3. Login testen** von einem anderen Rechner aus (nicht `pct enter`, das umgeht PAM/sshd):

```console
$ ssh -p <ssh-port> testuser@<ip>
```

## Dateien

```nix
# <repo-root>/modules/baseline/ldap.nix
{ ... }:
{
  services.sssd = {
    enable = true;
    config = ''
      [sssd]
      services = nss, pam
      domains = ldap

      [nss]

      [pam]

      [domain/ldap]
      id_provider = ldap
      auth_provider = ldap
      ldap_uri = <ldap-uri>
      ldap_search_base = <ldap-base-dn>
      ldap_schema = rfc2307
      cache_credentials = true
      enumerate = false
    '';
  };

  # LDAP-Nutzer stehen nie in users.users – ohne das hier bliebe ihr
  # $HOME beim ersten Login unangelegt.
  security.pam.services.sshd.makeHomeDir = true;
}
```

```diff
 # <repo-root>/modules/baseline/default.nix
 { ... }:
 {
   imports = [
     ./users.nix
     ./sudo.nix
     ./ssh.nix
     ./fail2ban.nix
+    ./ldap.nix
   ];
 }
```

> 💡 **Nice to know:** `services.sssd.enable = true` verdrahtet mehr automatisch, als es aussieht. NSS: Das Modul setzt `system.nssModules = [ pkgs.sssd ]` und trägt `sss` in `system.nssDatabases.passwd/group/shadow` ein, also fragt `/etc/nsswitch.conf` zusätzlich zu `/etc/passwd` auch sssd. PAM: `security/pam.nix` hängt für **jeden** Standard-PAM-Service (auch `sshd`) automatisch einen `pam_sss.so`-Eintrag in `auth`/`account`/`session` an (`enable = config.services.sssd.enable`) – nichts davon selbst konfigurieren. `users.mutableUsers = false` ändert daran nichts: Die Option schreibt nur `/etc/passwd`/`/etc/group` strikt aus `users.users` fest, die NSS-Quelle `sss` betrifft sie nicht. Quelle: [nixpkgs, `services/misc/sssd.nix`](https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/misc/sssd.nix), [nixpkgs, `security/pam.nix`](https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/security/pam.nix).

> ⚠️ Bind-Credentials im Klartext: Verlangt `<ldap-uri>` einen authentifizierten Bind, müssen `ldap_default_bind_dn` und `ldap_default_authtok` rein. `services.sssd.config` landet per `pkgs.writeText` im weltlesbaren Nix-Store – ein Passwort dort wäre für jeden im Container sichtbar. Die Option `services.sssd.environmentFile` umgeht das: In `ldap.nix` steht nur ein Platzhalter (`ldap_default_authtok = $SSSD_LDAP_DEFAULT_AUTHTOK`), der Klartextwert liegt in einer Datei außerhalb des Repos (z. B. `/etc/sssd-ldap-bind.env`, `chmod 600 root:root`), die `sssd` per `envsubst` erst beim Dienststart einsetzt. Das ist immer noch Klartext auf der Platte, nur nicht mehr im Store. Echtes Secrets-Management gibt es in Projekt 1 bewusst noch nicht (Exkurs in Schritt 15, vollständig ab Projekt 2 mit sops-age) – diese Lücke bleibt bis dahin offen. Quelle: [nixpkgs, `services.sssd.environmentFile`](https://github.com/NixOS/nixpkgs/blob/release-26.05/nixos/modules/services/misc/sssd.nix).

## Prüfen

- `getent passwd testuser` liefert im Container eine Zeile für den LDAP-Testnutzer (Beweis, dass NSS über `sss` auflöst).
- `ssh -p <ssh-port> testuser@<ip>` fragt nach einem **Passwort** (kein Key-Prompt) und lässt nach korrekter Eingabe einloggen.
- Nach dem ersten Login existiert `/home/testuser` mit dem Testnutzer als Eigentümer (`pam_mkhomedir` griff).
- `id testuser` zeigt die aus LDAP stammende UID/GID-Zuordnung.

## Wenn's schiefgeht

**`getent passwd testuser` liefert nichts, `journalctl -u sssd` zeigt wiederholte Verbindungsprobleme:** Meist scheitert der Bind – anonymes Lesen ist auf dem Server verboten. Mit dem `ldapsearch`-Befehl aus Schritt 1 gegenprüfen; falls nötig, Bind-DN/Passwort wie oben ergänzen.

**Nutzer löst per `getent` auf, aber `id testuser` zeigt keine Gruppen oder der Login scheitert trotz korrektem Passwort:** Wahrscheinlich Schema-Mismatch – der LDAP-Server nutzt `rfc2307bis` (Gruppenmitglieder per DN in `member`) statt des angenommenen `rfc2307` (`memberUid`). `ldap_schema = rfc2307bis` setzen und erneut testen.

**SSH fragt gar nicht erst nach einem Passwort, sondern bricht mit "Permission denied (publickey)" ab:** `PasswordAuthentication` ist in `ssh.nix` (Schritt 5) nicht aktiviert oder `UsePAM` wurde versehentlich auf `false` gesetzt – dann greift `pam_sss` nie. `services.openssh.settings.PasswordAuthentication = true;` prüfen.

## Rückweg

`./ldap.nix` aus `imports` in `default.nix` entfernen und rebuilden – `sssd` stoppt, LDAP-Nutzer verschwinden sofort aus `getent passwd`. Bereits angelegte Home-Verzeichnisse unter `/home/` bleiben als Datei-Leichen liegen (nicht von Nix verwaltet) und müssen bei Bedarf von Hand gelöscht werden; ebenso der sssd-Cache unter `/var/lib/sss/` (`rm -rf /var/lib/sss/db/*` vor einem Neuanfang).

## Querverweis

LDAP/sssd kommt in Teil I nicht vor – reines Projekt-II-Terrain. Direkt relevant ist aber Kapitel 6, „Nutzerverwaltung & SSH-Keys" ([06-alltagsbetrieb.md](../06-alltagsbetrieb.md)): Dort wird erklärt, was `users.mutableUsers = false` überhaupt einschränkt (`/etc/passwd`/`/etc/group`) – genau die Grenze, die LDAP-Nutzer über die separate NSS-Quelle `sss` umgehen.
