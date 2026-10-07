---
title: "Nur bauen, was Sie brauchen"
weight: 10
description: "Die Build-Tags, die Protokolle, Benutzerverzeichnisse, die Admin-API und den Provisioner aus dem Binary weglassen, und was jedes spart."
tags: [betrieb, build]
---

Jedes Protokoll liegt hinter einem Build-Tag, und ein Tag lässt es **ganz** weg: kein
Listener, kein Parser, keine Abhängigkeit, kein Code.

```sh
go install -tags nonfs,nowebdav,nosftp,nos3 github.com/go-fileshare/fileshare@latest   # SMB only
go build   -tags nosmb,nonfs,nosftp,nos3 .                                            # WebDAV only
```

Woher die **Personen** kommen, liegt hinter eigenen Tags: `nosql` lässt
die drei Datenbanktreiber weg, `noldap` den LDAP-Client.

| Build | Größe |
|---|---|
| alles | 31,2 MB |
| `-tags noldap` | 30,9 MB |
| `-tags nosftp` | 30,6 MB |
| `-tags noopenpubkey` (keine opkssh-Anmeldungen über SFTP) | etwa 2 MB weniger |
| `-tags nos3` | 31,1 MB |
| `-tags nonfs,nowebdav,nosftp,nos3` (nur SMB) | 28,1 MB |
| `-tags nopartitioned` (kein apfs, btrfs, xfs, zfs) | 29,2 MB |
| `-tags nosql` | 20,2 MB |
| `-tags nosql,noldap` | 19,8 MB |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp,nos3` | 16,2 MB |

Das sind die Zahlen aus der README von fileshare, gemessen, bevor die Admin-API
hinzukam; die Tabelle unten ist die spätere Messung.

## Die Admin-API und `nogrpc` {#the-admin-api-and-nogrpc}

Die [Admin-API]({{< relref "/administration/_index.md" >}}) hat gRPC und Protobuf mitgebracht, und
diese liegen hinter `nogrpc` (linux/amd64, gemessen am 29.09.2026, als die Zeilen oben
für alles auf 34,1 MB angewachsen waren):

| Build | Größe |
|---|---|
| alles, mit der Admin-API | 46,1 MB |
| `-tags nogrpc` | 34,4 MB |
| `-tags nosql,noldap,nogrpc` | 22,5 MB |

gRPC kostet **11,7 MB** – die [Plugin-Messung](#why-not-subprocess-plugins)
unten, erneut vorgenommen, jetzt, da es aus einem eigenen Grund hier ist. Eine Installation, die
ihre Freigaben in Dateien verwaltet, muss es nicht mitschleppen.

{{< callout type="warning" >}}
**Ein `admin`-Block in einem `nogrpc`-Build wird abgelehnt**

Statt ohne ihn bereitgestellt zu werden: Eine Konfiguration, die die API verlangt und
ohne sie startet, tut nicht, was sie sagt.
{{< /callout >}}

`nogrpc` lässt die Admin-API weg und mit ihr den
[Provisioner]({{< relref "/administration/volumes.md" >}}). [Health und Metriken]({{< relref "/administration/health.md" >}})
sind reines HTTP und bleiben.

`-tags noprovisioner` lässt nur den Provisioner weg, die privilegierte Rolle
`fileshare provisioner`, und mit ihr zfs, btrfs und projquota von go-fsctl:
Ein Server, der nie Volumes bereitstellt, muss den Code, der sie anlegt, nicht mitschleppen.
`fileshare provisioner` sagt in einem solchen Build genau das, statt „unknown command".

`nosql` ist mit Abstand der größte Hebel: PostgreSQL, MySQL und SQLite
wiegen zusammen **11,7 MB**, mehr als alle Protokolle dieses Programms
zusammen. Eine Installation, deren Benutzer in der Datei stehen, will es. Die Treiber werden
vom Kommando importiert und nicht von der Bibliothek, die sie verwendet; das macht die
Wahl zu einem Build-Tag statt zu einem Fork.

{{< callout type="info" >}}
**Ein Protokoll, ohne das dieses Binary gebaut wurde**

Eine Konfiguration, die eines nennt, erfährt *genau das*, statt „es gibt kein solches
Protokoll" – der Unterschied zwischen einem Tippfehler und einem Build-Tag. Dasselbe gilt
für `opkssh_client_id` unter `noopenpubkey` und einen `admin`-Block unter
`nogrpc`.
{{< /callout >}}

## Warum keine Plugins als Unterprozesse {#why-not-subprocess-plugins}

Es wurde **gemessen statt argumentiert**: `hashicorp/go-plugin` bringt gRPC und
Protobuf mit, die **allein 13,2 MB** kosten – mehr als dieses gesamte Binary
mit allen drei Protokollen und jedem Treiber darin. Ein Plugin-Host wäre doppelt
so groß, bevor er irgendetwas lädt, und jedes Plugin-Binary würde gRPC erneut
mitbringen.

Was die Angriffsfläche angeht, ist ein Tag auch das stärkere Werkzeug für alles, was Sie nicht
ausführen: **Code, der nie kompiliert wurde, ist nicht erreichbar**, ob in einer Sandbox oder nicht.

Was Tags *nicht* bieten, ist Isolierung zwischen den Protokollen, die Sie **tatsächlich** ausführen, noch
eine Möglichkeit, ein Protokoll ohne Neukompilieren hinzuzufügen. Beides ist real, und beides sind
Argumente für eine Prozessgrenze statt für ein kleineres Binary – und das ist
[ein Prozess pro Protokoll]({{< relref "/operations/isolation.md" >}}), mit genau diesem Binary, kein Plugin-Framework.
