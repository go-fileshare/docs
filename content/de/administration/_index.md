---
title: "Die Admin-API"
linkTitle: "Administration"
weight: 30
description: "Die gRPC-Admin-API, die Freigaben im laufenden Betrieb anlegt, ändert, deaktiviert und Berechtigungen vergibt, wo sie lauscht und was sie ablehnt."
tags: [administration, admin-api, grpc]
---

Seit v0.8.0 können sich die Freigaben ändern, während der Server läuft. Ein `admin`-Block
schaltet einen gRPC-Dienst ein, der Freigaben ohne Neustart und ohne Bearbeiten einer Datei
anlegt, ändert und offline nimmt.

```hcl
admin {
  listen       = "unix:///run/fileshare/admin.sock"   # made 0600
  state_file   = "/var/lib/fileshare/shares.json"
  source_roots = ["/srv/images", "/data"]
}
```

Nichts lauscht, solange der Block nicht geschrieben ist.

## Was sie tut {#what-it-does}

Der Dienst ist
[`fileshare.admin.v1.AdminService`](https://github.com/go-fileshare/fileshare/blob/v0.24.0/proto/fileshare/admin/v1/admin.proto):

| | |
|---|---|
| `CreateShare`, `UpdateShare`, `DeleteShare` | eine Freigabe aus einem Image, einem Verzeichnis oder (seit v0.21.0) einem Volume definieren; `read_only` oder `protocols` ändern (ihre Quelle kann sich nicht ändern: löschen Sie sie und legen Sie eine neue an) |
| `DisableShare`, `EnableShare` | eine Freigabe offline nehmen und wieder online bringen |
| `Grant`, `Revoke` | einem Subjekt Lese- oder Schreibzugriff geben oder ihn entziehen |
| `ListShares`, `GetShare` | jede Freigabe, mit dem, was sie bereitstellt, über welche Protokolle, und warum nicht über die anderen |
| `ListUsers`, `ListGroups` | wen eine Berechtigung nennen kann; jeder Benutzer mit den Protokollen, die seine Zugangsdaten beantworten können |
| `ReloadDirectory` | die Personen jetzt erneut einlesen – siehe [das Verzeichnis erneut einlesen]({{< relref "/administration/reload.md" >}}) |
| `GetServerInfo` | Name, Version, Startzeit, Generation und Listener |
| `ListParents`, `CreateVolume`, `ResizeVolume`, `SnapshotVolume`, `DeleteVolume`, `GetVolume`, `ListVolumes` | über einen privilegierten Provisioner angelegter Speicher und daraus bereitgestellte Freigaben – siehe [Volumes]({{< relref "/administration/volumes.md" >}}) (seit v0.21.0) |

Eine Berechtigung nennt einen **Benutzer**, eine **Gruppe** (`@group` in einer Konfigurationsdatei), einen
**`oidc:groups:`-Wert** oder einen **`oidc:user:`-Namen** – das Vokabular, das die
Konfigurationsdatei verwendet, siehe
[Personen, die der Identitätsanbieter nennt]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).

`grpc.health.v1` antwortet auf demselben Listener. `reflection = true` im Block
schaltet gRPC-Server-Reflection ein, für `grpcurl`.

### Jede Änderung sagt, was sie mit der Bereitstellung gemacht hat {#every-change-says-what-serving-it-did}

Eine Änderung antwortet mit einer `Applied`-Nachricht:

```proto
message Applied {
  uint64 generation = 1;          // the generation now being served
  uint64 connections_closed = 2;  // how many the previous one had open, and closed
}
```

Deshalb hat jeder RPC eine eigene Antwortnachricht, statt
die Freigabe selbst zurückzugeben: Die `.proto` folgt den `STANDARD`-Lint-Regeln von
[buf](https://buf.build), geprüft durch `buf lint` in der CI von fileshare, vor dem
„return the resource" von AIP-131 – die Hülle ist das, was `Applied` transportiert.

## Wo sie lauscht und wer sie aufrufen darf {#where-it-listens-and-who-may-call-it}

{{< callout type="error" >}}
**Über TCP gilt gegenseitiges TLS oder gar nichts**

`tls_cert_file`, `tls_key_file` und `client_ca_file`, alle drei,
**Loopback eingeschlossen**: Jeder lokale Benutzer kann Loopback erreichen, und diese API
entscheidet, wer wessen Dateien liest. Ein Unix-Socket wird mit **0600** angelegt, und seine
Berechtigungen sind seine Zugriffskontrolle.
{{< /callout >}}

```hcl
admin {
  listen         = "127.0.0.1:7443"
  tls_cert_file  = "/etc/fileshare/admin/server.pem"
  tls_key_file   = "/etc/fileshare/admin/server.key"
  client_ca_file = "/etc/fileshare/admin/clients.pem"
  state_file     = "/var/lib/fileshare/shares.json"
  source_roots   = ["/srv/images"]
}
```

Der Listener ist
[grpc-transports/control](https://github.com/grpc-transports/control). Jede
Änderung wird mit ihrem Urheber protokolliert: dem CN des Client-Zertifikats oder der uid des
Socket-Peers.

## Was sie anfasst und was nicht {#what-it-will-and-will-not-touch}

**Sie verwaltet die Freigaben, die sie angelegt hat.** Eine in der Konfiguration geschriebene Freigabe wird
mit denselben Feldern aufgelistet, und eine Änderung ihrer Definition wird mit
`FAILED_PRECONDITION` abgelehnt: Eine an zwei Stellen definierte Freigabe ist eine Frage, die niemand
beantworten will, und die Datei ist der Ort, an dem diese definiert ist. Die Freigaben der API liegen in
`state_file`, atomar geschrieben, und werden beim nächsten Start wieder bereitgestellt – deshalb
wird ein Block ohne `state_file` abgelehnt.

**Eine Quelle muss unter `source_roots` liegen**, aufgelöst – Links verfolgt, `..`
entfernt –, und es ist der aufgelöste Pfad, der gespeichert wird. Ohne `source_roots` kann die
API **überhaupt keine Freigaben anlegen**: Der Prozess kann `/dev` und `/etc` lesen,
und niemand wollte diese einem Aufrufer übergeben. Das wird bei jedem Start erneut geprüft,
und eine Freigabe in `state_file`, die nicht mehr unter einer Wurzel liegt, verhindert den Start
und wird dabei genannt.

**Jede Freigabe hat mindestens eine Berechtigung.** Eine Freigabe ohne ist für jeden offen, der sich
authentifiziert. Die Datei darf das absichtlich sagen; ein API-Aufruf sollte es nicht
durch Weglassen sagen. Also braucht `CreateShare` eine Berechtigung, und das Widerrufen der letzten wird
abgelehnt – löschen Sie stattdessen die Freigabe.

**Namen, die ein Protokoll transportieren kann.** Ein Freigabename hat höchstens 80 Zeichen, beginnt
und endet nicht mit einem Leerzeichen, ist nicht `.` oder `..` und enthält kein nicht druckbares
Zeichen und keines von `:*?"<>|{}%`. Subjekte enthalten ebenfalls kein nicht druckbares Zeichen,
und eine Freigabe nimmt höchstens 1000 Berechtigungen auf – lange vorher eine Aufgabe für eine Gruppe.
Jedes davon wurde vor v0.14.0 akzeptiert, in `state_file` geschrieben und war dann
bei jedem Start fatal.

**Keine Freigabe darf enthalten:** die Konfiguration, die Zustandsdatei oder die Geheimnisse,
die sie nennen: Wer hineinschreibt, würde umschreiben, wer was darf. Seit v0.17.0
gehören dazu `authorized_keys_file`, `dsn_file` und
`bind_password_file` eines `users`-Blocks, die `ca_file` des `ssf`-Blocks und eine SQLite-Datenbank, die ein DSN
nennt: Wer schreiben darf und seinen Schlüssel zu den `authorized_keys` eines anderen hinzufügen könnte, würde
sich als dieser anmelden. **Ebenso wenig darf eine Freigabe eine andere enthalten** (siehe
[Verzeichnisse]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})).

**Eine Änderung wird wie eine Konfiguration geprüft, geöffnet, festgeschrieben und dann
bereitgestellt.** Eine Änderung, die der Server nicht einhalten kann – ein Image, das sich nicht öffnen lässt, ein
Name, den niemand im Verzeichnis hat –, wird abgelehnt, wobei das Bereitgestellte und das
Geschriebene unverändert bleiben.

## Eine Freigabe deaktivieren {#disabling-a-share}

**Deaktivieren entspricht Sambas `available = no`**: Die Freigabe bleibt definiert, und jeder
Verbindungsversuch schlägt fehl; ihre offenen Verbindungen werden geschlossen, und ihr Image oder
Verzeichnis wird **losgelassen**, sodass die Datei ersetzt werden kann, während sie offline ist.
`EnableShare` stellt sie wieder bereit und wird abgelehnt, wenn ihre Quelle sich nicht mehr
öffnen lässt. `CreateShare` kann eine Freigabe auch bereits deaktiviert anlegen.

Anders als jede andere Änderung gilt sie auch für eine Freigabe **der Konfiguration** –
eine Freigabe offline zu nehmen ist ein Vorgang, keine Definition –, und sie übersteht einen
Neustart: Sie wird in der Zustandsdatei gespeichert. [`fileshare check`]({{< relref "/configuration/check.md" >}})
listet auf, was offline ist.

## Eine Änderung ist eine neue Generation {#a-change-is-a-new-generation}

{{< callout type="error" >}}
**Eine Änderung startet die Protokollserver neu und schließt ihre Verbindungen**

SMB prüft einmal pro Tree Connect, wer sich verbinden darf, und SFTP baut den
Baum einer Person einmal pro Anmeldung auf, sodass eine Sitzung, die einen Widerruf überdauert,
den gerade entzogenen Zugriff behielte. „Widerrufen, außer für alle, die schon
verbunden waren" ist kein Widerruf.
{{< /callout >}}

Eine Änderung wird also nicht *auf* die laufenden Server angewendet: Sie werden durch neue
ersetzt, die aus der neuen Liste gebaut werden – eine **Generation** –, so wie es ein Neustart täte,
ohne die Ports aufzugeben. Die Bibliotheken könnten es nicht anders:
go-filesystems/smb hat `Share` und kein `Unshare`, nfs hat `Export` und kein
`Unexport`.

Die Ports bleiben gebunden, sodass ein Client, der sich während einer Änderung verbindet, **wartet**, statt
abgelehnt zu werden. Images, deren Freigabe sich nicht geändert hat, behalten ihren Treiber und werden nie
ein zweites Mal geöffnet. Clients verbinden sich neu; das ist der Preis, gezahlt bei jeder
Änderung – deshalb wendet die API eine Änderung pro Aufruf an statt einer pro
Feld.

## Grenzen {#limits}

- `--isolate` passt noch nicht zu einem `admin`-Block – ebenso wenig zu einem `metrics`-Block –:
  Es gibt keinen einzelnen Prozess, auf den eine Änderung angewendet werden könnte. Die Kombination wird
  abgelehnt.
- Ein mit `-tags nogrpc` gebautes Binary hat keine Admin-API, und eine Konfiguration mit
  einem `admin`-Block wird **abgelehnt**, statt ohne sie bereitgestellt zu werden. Dieses Tag spart
  11,7 MB; siehe [nur bauen, was Sie brauchen]({{< relref "/operations/build-tags.md#the-admin-api-and-nogrpc" >}}).

## Die `.proto` ändern {#changing-the-proto}

Der Go-Code unter `proto/` wird generiert und committet, sodass `go install` kein
`protoc` braucht. Nach einer Änderung an einer `.proto` – der der Admin-API oder der des Provisioners,
`proto/fileshare/provision/v1/provision.proto` – generieren Sie beide mit den
Versionen neu, die die CI von fileshare pinnt (protoc 34.1, protoc-gen-go v1.36.12,
protoc-gen-go-grpc v1.6.2). Die CI generiert sie neu und schlägt fehl, wenn der committete
Code abweicht oder wenn generierter Code nicht committet wurde:

```sh
protoc -I proto --go_out=. --go_opt=module=github.com/go-fileshare/fileshare \
  --go-grpc_out=. --go-grpc_opt=module=github.com/go-fileshare/fileshare \
  proto/fileshare/admin/v1/admin.proto proto/fileshare/provision/v1/provision.proto
```
