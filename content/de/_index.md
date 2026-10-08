---
title: "go-fileshare"
linkTitle: "Startseite"
type: docs
cascade:
  type: docs
description: "go-fileshare stellt Disk-Images und Verzeichnisse über SMB, NFS, WebDAV, SFTP und S3 bereit, mit einem einzigen Satz von Benutzern und Regeln pro Freigabe aus einer einzigen Konfiguration."
---

**Ein Disk-Image – oder ein Verzeichnis –, bereitgestellt über SMB, NFS, WebDAV, SFTP und S3:
dieselben Benutzer, derselbe Zugriff pro Freigabe, aus einer einzigen Konfigurationsdatei.** Reines
Go, `CGO_ENABLED=0`, ein Binary. Diese Seiten beschreiben
[v0.24.0]({{< relref "/status.md" >}}).

```sh
go install github.com/go-fileshare/fileshare@latest

fileshare --image disk.img --user alice --password-file pw   # one image, now
fileshare --config /etc/fileshare.d                          # several, with users
```

Das Passwort kommt aus einer **Datei**, nie aus einem Flag: Ein Argument ist in der
Prozessliste für jeden Benutzer der Maschine sichtbar.

## Warum ein einziges Programm {#why-one-program}

Welches Protokoll ein Image transportiert, hängt vom Client am anderen Ende ab:
macOS und Windows greifen zu SMB, eine Linux-Flotte hat bereits NFS, ein Browser oder ein
Telefon hat HTTP. Drei Server zu betreiben, jeden mit eigener Konfigurationsdatei und
eigener Vorstellung davon, wer `alice` ist, ist ein sicherer Weg, zwei davon auf subtile Weise falsch zu konfigurieren.

Deshalb gibt es eine Konfiguration, einen Satz von Benutzern, einen Satz von Regeln pro Freigabe –
und die Protokolle sind Listener darüber.

## Was es beim Start ausgibt {#what-it-prints-at-startup}

```
smb    on 0.0.0.0:445 — photos and scratch
webdav on 0.0.0.0:8080 — photos and scratch
sftp   on 0.0.0.0:2222 — photos and scratch
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

Dieser letzte Absatz zeigt die Form des ganzen Programms: Eine Regel, die die Konfiguration
verlangt, und ein Protokoll, das sie nicht einhalten kann, treffen sich nicht stillschweigend in der Mitte.
Siehe [Was ein Protokoll zusichern kann]({{< relref "/protocols/_index.md" >}}).

## Wie es weitergeht {#where-to-go-next}

| | |
|---|---|
| [Die Konfiguration]({{< relref "/configuration/_index.md" >}}) | Benutzer, Gruppen, Freigaben, `serve`-Blöcke |
| [Freigaben: Images, Geräte, Verzeichnisse]({{< relref "/configuration/shares.md" >}}) | was erkannt wird, was deklariert werden muss |
| [Benutzer, Gruppen und Verzeichnisse]({{< relref "/configuration/identity.md" >}}) | Dateien, SQL, LDAP und was jede Quelle beweisen kann |
| [`check`]({{< relref "/configuration/check.md" >}}) | die ganze Konfiguration vor einem Neustart zurücklesen |
| [Protokolle]({{< relref "/protocols/_index.md" >}}) | was jedes zusichern kann und was nicht |
| [Die Admin-API]({{< relref "/administration/_index.md" >}}) | Freigaben anlegen, Berechtigungen vergeben und Freigaben offline nehmen, ohne Neustart |
| [Volumes]({{< relref "/administration/volumes.md" >}}) | ZFS-Datasets, btrfs-Subvolumes und XFS/ext4-Projekt-Quotas, über die API angelegt und bereitgestellt |
| [Health und Metriken]({{< relref "/administration/health.md" >}}) | `/healthz`, `/readyz`, `/metrics` |
| [Das Verzeichnis erneut einlesen]({{< relref "/administration/reload.md" >}}) | `reload`, `SIGHUP` und was eine Entfernung mit offenen Sitzungen macht |
| [TLS]({{< relref "/security/tls.md" >}}) | Dateien oder ACME; und warum WebDAV unverschlüsselt abgelehnt wird |
| [Was was widerruft]({{< relref "/security/_index.md" >}}) | KRL, CRL, Shared Signals – und warum jedes geschlossen fehlschlägt (fail closed) |
| [Nur bauen, was Sie brauchen]({{< relref "/operations/build-tags.md" >}}) | ein Build-Tag lässt ein Protokoll ganz weg |
| [Ein Prozess pro Protokoll]({{< relref "/operations/isolation.md" >}}) | `--isolate` |
| [Stand]({{< relref "/status.md" >}}) | was geprüft ist und was noch fehlt |

## Lizenz {#licence}

BSD-3-Clause.
