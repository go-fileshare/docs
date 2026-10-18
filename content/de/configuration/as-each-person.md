---
title: "Verzeichnisse unter der Identität jeder Person"
weight: 15
description: "identity = \"system\": eine Verzeichnisfreigabe, deren Dateien ein Worker öffnet, der als die angemeldete Person läuft; der Kernel wendet die Rechte des Baums an, und was sie anlegt, gehört ihr."
tags: [konfiguration, freigaben, identität, sicherheit]
---

Gewöhnlich bedient ein einziger fileshare-Prozess alle unter einem einzigen
Konto: fileshare entscheidet, wer was darf, und jede Datei, die er anlegt,
gehört diesem Konto. Eine Verzeichnisfreigabe kann stattdessen **unter der
Identität jeder Person** bedient werden. Die Dateien der angemeldeten Person
öffnet ein Prozess mit ihrer uid, ihrer gid und ihren Gruppen: Der Kernel
wendet die Rechte des Baums an, und was sie anlegt, gehört ihr.

```hcl
identity {
  service_user = "fileshare"   # das Konto, unter dem die Protokolle laufen
  min_uid      = 1000          # der Standard; uid 0 wird immer abgelehnt
  min_gid      = 1000
  deny_uids    = [65534]
  idle         = "2m"          # wie lange ein unbenutzter Worker bleibt
}

share "home" {
  directory = "/srv/home"
  identity  = "system"
}
```

## Drei Prozesse {#three-processes}

`fileshare serve` muss **als root gestartet** werden. Root bleibt es nur, um
den Worker jeder Person zu starten. Alles, was dem Netz zugewandt ist, läuft
in einem zweiten Prozess, als `service_user`.

| Prozess | läuft als | tut |
|---|---|---|
| der **Starter** (`fileshare serve`) | root | bindet die Ports, startet das Front, startet Worker, wenn das Front darum bittet. Er spricht kein Protokoll. |
| das **Front** | `service_user`, mit den Gruppen dieses Kontos | spricht die Protokolle und authentifiziert, wie `serve` es immer getan hat. Für eine solche Freigabe erbittet es vom Starter eine **Person und eine Freigabe**, nie eine uid oder einen Pfad. |
| ein **Worker** | die Person | bedient eine Freigabe für das Front, über einen Socket, den der Starter übergeben hat |

Ein Worker prüft zuerst, dass er genau als die erwartete uid, gid und Gruppen
läuft und dass `setuid(0)` scheitert. Erst dann öffnet er den Baum, über ein
`os.Root`. Ein Worker bedient alle Verbindungen dieser Person zu dieser
Freigabe, über SFTP wie über WebDAV, und endet, wenn ihn `idle` lang niemand
benutzt hat. Endet der Server, enden die Worker mit ihm.

## Wer eine Person auf diesem System ist {#who-a-person-is-on-this-system}

Der Starter löst die Person **selbst** auf, in dieser Reihenfolge:

1. **Das Verzeichnis**, wenn es `uidNumber` und `gidNumber` trägt:
   - die `attributes = { uidNumber = "2001", gidNumber = "2001" }` eines `user`-Blocks;
   - eine LDAP-Quelle mit `attributes = ["uidNumber", "gidNumber"]`;
   - die `users_attributes`-Abfrage einer SQL-Quelle.

   Die Zusatzgruppen sind die Gruppen, die die Person auflisten (`members`
   eines `group`-Blocks, `memberUid`), und die, die die Person nennt
   (`memberOf`), jede über ihre `gidNumber`. Eine Gruppe ohne sie ist keine
   Systemgruppe.
2. **`/etc/passwd` und `/etc/group`**, nach Namen.

**Abgelehnt**: uid 0, gid 0, `(uid_t)-1`, eine uid unter `min_uid`, eine
primäre gid unter `min_gid` und jede uid in `deny_uids`. Eine Zusatzgruppe
unter `min_gid` wird weggelassen, und das Protokoll sagt es. Eine Person, die
nirgends gefunden wird, wird ebenfalls abgelehnt: Es gibt kein gemeinsames
Ersatzkonto. Eine Person, für die der Identitätsanbieter bürgt, erhält einen
Worker nur mit [`local_names`]({{< relref "/configuration/identity.md" >}}),
das ihren Namen zu einem lokalen macht.

Ein übernommenes Front kann also nur eine Person erbitten, die existiert und
erlaubt ist, auf einer Freigabe, die existiert. **Wer** sich authentifiziert
hat, entscheidet weiterhin das Front, und der Starter kann das nicht für es
prüfen.

## Was ein Client sieht {#what-a-client-sees}

| | SFTP | WebDAV |
|---|---|---|
| ein Worker wird genommen | bei der Anmeldung, für die ganze Verbindung | pro Anfrage; Sperren leben so lange wie der Worker |
| der Starter lehnt die Person ab | die Freigabe fehlt in ihrem Baum (ein leerer Baum ist eine abgelehnte Anmeldung) | `403` |
| kein Worker kann starten | ebenso | `503` |

Der Grund geht ins Protokoll des Servers, nie an den Client.

{{< callout type="warning" >}}
**Noch nicht über SMB, NFS oder S3**

Sie lehnen eine solche Freigabe ab, und [`check`]({{< relref "/configuration/check.md" >}})
sagt es. SMB und NFS brauchen in ihren Bibliotheken ein Dateisystem pro
Sitzung. NFS wird es nur unter Kerberos tun, denn `AUTH_UNIX` ist die bloße
Behauptung des Clients (RFC 5531 §14).
{{< /callout >}}

## Was es vom System verlangt {#what-it-asks-of-the-system}

- **Unix**: Linux, die BSDs und macOS. Unter Windows wird die Freigabe abgelehnt.
- **Lesbar für `service_user`**: die Konfiguration und jede Datei, die sie
  nennt (Passwortdateien, Host-Schlüssel, Zertifikate), denn das Front liest
  sie.
- **Ausführbar für die Personen**: das Programm, denn jeder Worker führt es
  als seine Person aus.
- **Ein Neustart**, um eine solche Freigabe hinzuzufügen oder zu ändern. Der
  Starter liest die Konfiguration einmal und lehnt eine Freigabe ab, die er
  nicht kennt, statt das Front sie bedienen zu lassen.
- `--isolate` geht nicht damit zusammen.

Der Entwurf und der Grund jeder Entscheidung stehen in
[docs/identity.md](https://github.com/go-fileshare/fileshare/blob/main/docs/identity.md).
