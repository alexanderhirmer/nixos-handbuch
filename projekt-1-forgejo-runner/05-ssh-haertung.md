---
title: "SSH-Härtung"
weight: 5
---

# Schritt 5: SSH läuft auf Port 40, `PermitRootLogin` ist deaktiviert

## Ziel

`sshd` hört nur noch auf Port `<ssh-port>`, Root-Login ist verboten, und trotzdem bleibt Passwort-Login für die künftigen LDAP-Nutzer möglich – nur `<admin-user>` ist strukturell auf reinen Key-Login beschränkt.

## Voraussetzung

Schritt 3 (`<admin-user>` existiert, ausschließlich mit SSH-Key, ohne jedes Passwort-Feld) und Schritt 4 (passwortloses `sudo` für `<admin-user>`) sind abgeschlossen. `modules/baseline/default.nix` importiert bereits `./users.nix` und `./sudo.nix`.

## Durchführung

**1. Modul anlegen** (Inhalt siehe Abschnitt "Dateien") und **2. `default.nix` um eine Zeile erweitern** (Diff siehe "Dateien").

**3. Warum `PasswordAuthentication` global an bleiben muss:** `<admin-user>` wurde in Schritt 3 ohne `hashedPassword`, `password`, `initialPassword` oder `hashedPasswordFile` angelegt. Laut der `allowsLogin`-Prüfung in Nixpkgs' `users-groups.nix` gilt ein Konto ohne einen dieser Werte (Default `hashedPassword = null`) als gesperrt. Ob `PasswordAuthentication` serverseitig erlaubt ist, spielt für dieses eine Konto also keine Rolle: Es hat schlicht kein Passwort, das PAM prüfen könnte. Für die LDAP-Nutzer aus Schritt 8 gilt das nicht – dafür muss `PasswordAuthentication` (und `KbdInteractiveAuthentication`, falls sssd darüber statt über die klassische Passwortabfrage geht) global erlaubt bleiben. Ein `Match`-Block für `<admin-user>` wäre technisch möglich – `extraConfig` landet laut Quellcode von `sshd.nix` immer *hinter* den aus `settings` generierten Zeilen, also syntaktisch an der einzig gültigen Stelle. Trotzdem verzichtet dieses Modul bewusst darauf: Mehrere NixOS-Issues beschreiben genau `Match` + `PasswordAuthentication` + PAM als unzuverlässig ([#12867](https://github.com/NixOS/nixpkgs/issues/12867), offen/"stale"; [#18503](https://github.com/NixOS/nixpkgs/issues/18503), geschlossen, ohne belegten Fix). Die fehlende `hashedPassword` ist der robustere Hebel.

**4. Testen, dann anwenden:**

```console
$ nixos-rebuild build --flake /etc/nixos#<hostname>
$ nixos-rebuild switch --flake /etc/nixos#<hostname>
```

> 💡 **Nice to know:** `sshd` läuft unter NixOS mit `Type = "notify-reload"`; die Unit wird bei Änderungen an `sshd_config` neu geladen (`restartTriggers`), nicht hart gekillt. Die *bestehende* SSH-Sitzung, über die `switch` läuft, bleibt deshalb offen – sie nutzt den längst aufgebauten TCP-Socket weiter. Nur *neue* Verbindungen brauchen ab sofort `-p 40`. Schließe die aktuelle Sitzung trotzdem nicht, bevor eine neue Verbindung auf Port 40 nachweislich funktioniert – und falls doch etwas schiefgeht: `pct enter <vmid>` (Schritt 1) bleibt der Rettungsanker, weil er nicht über `sshd` läuft.

> ⚠️ Ungeprüft: Ob das Verhalten aus den verlinkten Issues auf dem in NixOS 26.05 gebündelten OpenSSH noch exakt so reproduzierbar ist, wurde nicht nachgestellt – die Issues selbst sind aber real und beschreiben genau dieses Muster.

## Dateien

`<repo-root>/modules/baseline/ssh.nix` (neu):

```nix
# <repo-root>/modules/baseline/ssh.nix
{ ... }:
{
  services.openssh = {
    enable = true;
    ports = [ 40 ]; # <ssh-port>
    openFirewall = true;
    settings = {
      PermitRootLogin = "no";
      # Global an, weil LDAP-Nutzer (Schritt 8) per Passwort einloggen.
      # <admin-user> hat kein Passwort gesetzt und kommt darüber ohnehin nie rein.
      PasswordAuthentication = true;
      KbdInteractiveAuthentication = true;
    };
  };
}
```

`<repo-root>/modules/baseline/default.nix` (geändert, eine Zeile):

```nix
  imports = [
    ./users.nix
    ./sudo.nix
+   ./ssh.nix
  ];
```

Die konkrete Firewall-Freigabe für Port 40 wird hier vorerst über `openFirewall = true` erledigt; Schritt 9 formalisiert die Firewall-Regeln vollständig (`firewall.nix`).

## Prüfen

- `sudo sshd -T -f /etc/ssh/sshd_config | grep -Ei '^(port|permitrootlogin|passwordauthentication) '` muss `port 40`, `permitrootlogin no` und `passwordauthentication yes` zeigen.
- Eine **neue** Verbindung `ssh -p 40 <admin-user>@<ip>` gelingt ohne Passwortabfrage (Key).
- `ssh -p 40 root@<ip>` wird abgelehnt (`Permission denied`, unabhängig von der versuchten Methode).
- Ein Verbindungsversuch auf dem alten Port (`ssh -p 22 …@<ip>`) schlägt fehl, weil dort niemand mehr lauscht.

## Wenn's schiefgeht

**"ssh: connect to host \<ip\> port 22: Connection refused":** Erwartetes Verhalten nach dem Switch, kein Fehler – `sshd` lauscht nur noch auf 40. Fix: `-p 40` an den `ssh`-Aufruf anhängen.

**Kompletter SSH-Zugriffsverlust (Tippfehler im Port, Firewall verweigert 40 doch):** Über `pct enter <vmid>` auf dem Proxmox-Host einsteigen (läuft nicht über `sshd`), `ssh.nix` korrigieren, `nixos-rebuild switch --flake /etc/nixos#<hostname>` erneut von innen ausführen.

**Nach eigenem `Match`-Block plötzlich unklares Passwortverhalten (Passwort wird verlangt, Login schlägt trotzdem fehl):** Deckt sich mit den oben verlinkten NixOS-Issues zu `Match` + PAM. Fix: `Match`-Block wieder entfernen, stattdessen wie hier beschrieben auf das fehlende Passwort von `<admin-user>` verlassen.

## Rückweg

`./ssh.nix` aus der `imports`-Liste in `default.nix` entfernen (oder darin `ports = [ 22 ];` und `PermitRootLogin = "prohibit-password";` setzen, den Nixpkgs-Default) und erneut `nixos-rebuild switch --flake /etc/nixos#<hostname>` ausführen. Bei Lockout wie oben: `pct enter <vmid>`.

## Querverweis

Teil I, Kapitel 6 ("Alltagsbetrieb") führt das Muster `services.openssh` mit `settings` und `openFirewall` bereits ein und begründet dort auch, warum konsequent `services.openssh.*` statt `services.sshd.*` verwendet wird.
