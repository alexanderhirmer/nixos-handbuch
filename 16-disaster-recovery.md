---
title: "Disaster Recovery"
weight: 16
---

# Disaster Recovery

## Lernziele

- Du kannst erklären, warum "Neuaufbau aus dem Repo" NixOS' Grundversprechen ist – und wo genau die Grenze liegt.
- Du unterscheidest, was NixOS automatisch absichert, von dem, was es nicht absichert.
- Du kannst eine grobe Backup-Strategie für Zustand skizzieren.
- Du ordnest den Impermanence-Ansatz ein – Konzept, nicht Pflichtprogramm.
- Du hast praktisch geübt: eine Maschine "zerstören" und aus dem Repo wiederherstellen.

## Warum das wichtig ist

Dieses Buch endet dort, wo Betrieb in der Realität eigentlich erst anfängt: Was passiert, wenn eine Maschine physisch oder virtuell einfach weg ist? NixOS' Versprechen – "baue mich aus dem Repo neu auf" – ist mächtig, deckt aber nur einen Teil des Problems ab.

## Neuaufbau aus dem Repo: das Versprechen

`configuration.nix`/`flake.nix` plus Git-Historie ist eine vollständige, versionierte Beschreibung des *Systems*. Mit den Werkzeugen aus Kapitel 4 und 14 (`nixos-anywhere`, `disko`) lässt sich eine neue – oder dieselbe, komplett platt gemachte – Maschine binnen Minuten wieder in exakt denselben Systemzustand bringen: dieselben Pakete, dieselben Dienste, dieselbe Konfiguration, ohne dass sich irgendjemand erinnern muss, was damals alles manuell installiert wurde.

## Was NixOS absichert – und was nicht

**Abgesichert:** Systemkonfiguration, installierte Pakete, Dienste samt deren Einstellungen – alles, was `nixos-rebuild` aus deiner Konfiguration erzeugt.

**Nicht abgesichert: Nutzdaten.** Eine Datenbank, hochgeladene Dateien, echte `/home`-Inhalte, der aktuelle Zustand von TLS-Zertifikaten, Logs – all das lebt außerhalb des Nix Store und außerhalb der Konfiguration, selbst wenn der *Dienst*, der sie erzeugt, vollständig deklarativ konfiguriert ist. Ein `nixos-anywhere`-Neuaufbau bringt dir einen frisch konfigurierten PostgreSQL-Server zurück – aber nicht die Datenbankinhalte von vorher. Das ist keine Einschränkung von NixOS, sondern liegt in der Natur der Sache: Nutzdaten sind per Definition das, was *nicht* aus einer Beschreibung reproduzierbar ist.

## Backup-Strategie für Zustand

NixOS-agnostisches, aber notwendiges Terrain: klassisches Backup-Tooling (z. B. restic oder borgbackup – hier nur als Beispiele genannt, keine Tiefenbehandlung, das ist eigenständiges Thema) für alles unter `/var/lib/<dienst>`, wo möglich Datenbank-Dumps statt Roh-Dateien im laufenden Betrieb sichern, und – der Punkt, der am häufigsten übersprungen wird – **Restores tatsächlich testen**. Ein Backup, das nie zurückgespielt wurde, ist bestenfalls eine Vermutung.

## Impermanence: der fortgeschrittene Ansatz

Ursprünglich von Graham Christensens Blogpost "Erase your darlings" popularisiert: Statt Zustand nur im Katastrophenfall zurückzuspielen, wird das Root-Dateisystem bei **jedem** Boot verworfen – Config-Drift (Kapitel 1) wird damit nicht nur verhindert, sondern aktiv unmöglich gemacht, weil jede manuelle, nicht deklarierte Änderung garantiert den nächsten Neustart nicht übersteht.

Zwei Umsetzungswege:

1. **tmpfs als Root** – die einfachste Variante, `/` lebt komplett im Arbeitsspeicher und ist nach jedem Boot leer. Risiko: Ein Stromausfall bedeutet Verlust von allem, was gerade nicht explizit persistiert war.
2. **ZFS-/Btrfs-Snapshot-Rollback** – `/` liegt auf einem echten Dateisystem, wird aber bei jedem Boot auf einen einmalig angelegten, leeren Snapshot zurückgerollt. Übersteht auch Stromausfälle sauber. Das Original-Muster, direkt aus Christensens Blogpost:

```nix
{
  boot.initrd.postDeviceCommands = lib.mkAfter ''
    zfs rollback -r rpool/local/root@blank
  '';
}
```

In beiden Fällen übernimmt das Community-Modul [impermanence](https://github.com/nix-community/impermanence) die "Ausnahmeliste" – was explizit erhalten bleiben soll:

```nix
environment.persistence."/persist" = {
  directories = [ "/etc/ssh" "/var/log" "/var/lib/postgresql" ];
  files = [ "/etc/machine-id" ];
};
```

`/etc/ssh` steht hier nicht zufällig an erster Stelle: Ohne persistierte Host-Keys bekommt die Maschine bei jedem Boot neue – mit Konsequenzen, die im Abschnitt "Typische Fehler" gleich folgen.

## Vollständiges Beispiel

Die Disaster-Recovery-Übung dieses Kapitels in Kurzform, aufbauend auf Kapitel 4/14:

```console
# Katastrophe simulieren: Buch-VM komplett platt machen
# (in der Praxis: VM löschen und neu anlegen, oder Platte neu partitionieren)

# Neuaufbau ausschließlich aus dem Repo
$ nix run github:nix-community/nixos-anywhere -- \
    --flake .#buch-vm root@<neue-ip>

# Prüfen, was fehlt, das nicht im Repo stand
$ ssh alex@<neue-ip>
$ ls /var/lib/   # z. B. Datenbankinhalte – falls nicht separat gesichert, jetzt leer
```

Der Unterschied zwischen "Konfiguration wiederhergestellt" (sofort, aus dem Repo) und "Daten wiederhergestellt" (nur, wenn ein separates Backup existiert) wird an diesem Punkt sehr konkret.

> 💡 **Nice to know:** Vor einem Reboot lässt sich bei der ZFS-Variante genau nachsehen, was verloren ginge: `zfs diff rpool/local/root@blank` zeigt die Differenz zum leeren Ausgangs-Snapshot – nützlich, um vor dem ersten produktiven Einsatz zu prüfen, ob wirklich nur Unwichtiges verschwindet.

> 💡 **Nice to know:** impermanence ist bewusst ein eigenständiges Community-Projekt, kein Teil von NixOS selbst – ähnlich wie Home Manager (Kapitel 13). Es lohnt sich erst, wenn der Aufwand (Persistenzliste pflegen, Backup-Strategie für `/persist` selbst) den Nutzen (garantierte Config-Drift-Freiheit) tatsächlich aufwiegt – für einen einzelnen Hobby-Server oft übertrieben, für eine Flotte gleichartiger Maschinen (Kapitel 14) oft genau richtig.

## Typische Fehler

**1. Datenverzeichnis eines Diensts nicht persistiert:**

```
FATAL: data directory "/var/lib/postgresql/16" does not exist
```

*Ursache:* Der Pfad fehlt sowohl in `environment.persistence` als auch in einem separaten Backup – nach einem Neuaufbau oder Reboot ist er schlicht leer bzw. nicht vorhanden.
*Fix:* Pfad zur Persistenzliste hinzufügen (bei Impermanence) bzw. ins Backup-Konzept aufnehmen (beim klassischen Neuaufbau).

**2. `/etc/ssh` nicht persistiert:**

```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

– bei jedem einzelnen Neustart, auf jedem Client, der sich schon einmal verbunden hat.

*Ursache:* Ohne persistierte Host-Keys erzeugt OpenSSH bei jedem Boot neue – aus Sicht der Clients sieht das wie ein Man-in-the-Middle-Angriff aus.
*Fix:* `/etc/ssh` in `environment.persistence` aufnehmen (siehe Beispiel oben).

## Übung

1. Simuliere in deiner Buch-VM eine Katastrophe: Zerstöre die VM (löschen und neu anlegen, oder Platte neu partitionieren) und baue sie ausschließlich aus deinem Git-Repository wieder auf, mit den Werkzeugen aus Kapitel 4/14. Notiere dir, was du dabei verloren hast, das nicht im Repo stand.
2. *(Fortgeschritten, optional)* Richte impermanence mit tmpfs-Root in einer separaten Test-VM ein, boote zweimal neu, und bestätige, dass eine bewusst nicht persistierte Testdatei tatsächlich verschwunden ist.

**Lösungsskizze:**

Zu 1: Alles, was nicht in `configuration.nix`/`flake.nix` steht und nicht separat gesichert wurde, ist weg – typischerweise Nutzdaten in `/var/lib/<dienst>` und ggf. `/home`-Inhalte.

Zu 2: Nach dem zweiten Reboot sollte die Testdatei verschwunden sein; taucht sie noch auf, war sie versehentlich doch persistiert (oder das Root-Dateisystem ist gar nicht wirklich tmpfs).

## Zusammenfassung

- NixOS sichert die Systemkonfiguration automatisch ab – Nutzdaten sind explizit *nicht* enthalten und brauchen eine eigene Backup-Strategie.
- Ein Backup zählt erst, wenn der Restore tatsächlich getestet wurde.
- Impermanence verwirft das Root-Dateisystem bei jedem Boot (tmpfs oder ZFS-/Btrfs-Snapshot-Rollback) und macht Config-Drift damit strukturell unmöglich, nicht nur unwahrscheinlich.
- Das `impermanence`-Modul verwaltet die Ausnahmeliste dessen, was explizit erhalten bleiben soll – `/etc/ssh` gehört dort so gut wie immer hinein.
- `zfs diff pool/root@blank` zeigt vor einem Reboot, was verloren ginge.
- Der praktische Test – Maschine zerstören, aus dem Repo wiederherstellen – zeigt zuverlässiger als jede Theorie, was tatsächlich abgesichert ist.

## Verwendete Befehle/Optionen mit Quelle

| Befehl/Option | Quelle |
|---|---|
| `environment.persistence."<pfad>"` (`directories`/`files`) | [nix-community/impermanence](https://github.com/nix-community/impermanence), [hanckmann.com – Nixos and Erasing My Darlings](https://hanckmann.com/posts/20230104-nixos-and-erasing-my-darlings/) |
| `zfs rollback -r pool/root@blank`, `boot.initrd.postDeviceCommands` | [Graham Christensen – Erase your darlings](https://grahamc.com/blog/erase-your-darlings/) |
| `zfs diff pool/root@blank` | [Graham Christensen – Erase your darlings](https://grahamc.com/blog/erase-your-darlings/) |
| tmpfs-als-Root-Variante | [Elis Hirwing – NixOS: tmpfs as root](https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/) |
| Langzeit-Praxisbericht (3 Jahre Impermanence) | [b.tuxes.uk – Three Years of Ephemeral NixOS](https://b.tuxes.uk/three-years-of-ephemeral-nixos.html) |
| `nixos-anywhere` (Neuaufbau) | Community-Projekt, siehe Kapitel 4/14 |
| SSH-Host-Key-Warnung, PostgreSQL-Fehlermeldung | Standard-Fehlermeldungen der jeweiligen Software (allgemeines Wissen) |
