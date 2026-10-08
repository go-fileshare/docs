---
title: "Volumes: Speicher, den fileshare anlegt"
weight: 10
description: "ZFS-, btrfs-, XFS- und ext4-Speicher über den privilegierten fileshare provisioner anlegen und Freigaben daraus bereitstellen."
tags: [administration, volumes, zfs, btrfs, quotas]
---

Seit v0.21.0 kann die [Admin-API]({{< relref "/administration/_index.md" >}}) **den Speicher anlegen, den eine Freigabe
bereitstellt** – ein **ZFS-Dataset**, ein **btrfs-Subvolume**, ein **XFS- oder ext4-Verzeichnis
unter einer Projekt-Quota** –, mit Größe, Eigentümer und Buchführung, und eine Freigabe daraus bereitstellen.
Nur Linux. Ceph wird noch nicht unterstützt.

Diesen Speicher anzulegen erfordert `CAP_SYS_ADMIN` (quotactl, die btrfs-Quota-ioctls,
`/dev/zfs`, mount(2)). `fileshare serve` parst fünf Netzwerkprotokolle für
Fremde und hält kein Privileg, also ist der Teil, der eines braucht, eine zweite Rolle desselben
Binarys, `fileshare provisioner`, hinter einem Unix-Socket und einem kleinen, geschlossenen
Protokoll:

```
                         admin gRPC (unix socket 0600 + peer uid, or mTLS)
  operator ─────────────────────────────────────────────▶  fileshare serve
                                                             (unprivileged)
                                                                  │
                     provision gRPC (unix socket, SO_PEERCRED = fileshare's uid)
                                                                  ▼
                                                    fileshare provisioner
                                                    (CAP_SYS_ADMIN, CAP_CHOWN, CAP_FOWNER)
                                                                  │
                                    go-fsctl: zfs · btrfs · projquota (ioctls, no CLI)
```

Ein Binary statt zweier Daemons: eine Version zum Ausrollen, eine
Protokolldefinition. Den Speicher selbst übernimmt [go-fsctl](https://go-fsctl.github.io/)
(reines Go, kein `zfs`- oder `btrfs`-Kommando). Das Design und was sich während
des Baus geändert hat, stehen im fileshare-Repository unter
[`docs/volumes.md`](https://github.com/go-fileshare/fileshare/blob/v0.24.0/docs/volumes.md).

## Beide Prozesse, konfiguriert {#both-processes-configured}

`fileshare serve`, als Benutzer `fileshare` (uid 990):

```hcl
# /etc/fileshare/fileshare.hcl
admin {
  listen          = "unix:///run/fileshare/admin.sock"
  state_file      = "/var/lib/fileshare/shares.json"
  source_roots    = ["/srv/fileshare/volumes"]
  provisioner     = "unix:///run/fileshare-provisioner/provisioner.sock"
  provisioner_uid = 0          # who must answer on that socket; 0 is the default
  allowed_uids    = [990]      # who may call this API
}
```

| Schlüssel | |
|---|---|
| `provisioner` | der Socket des Provisioners, `unix:///path`. Ohne ihn antwortet jeder Volume-Aufruf mit `FAILED_PRECONDITION`, und keine Freigabe wird aus einem Volume erstellt |
| `provisioner_uid` | die uid, unter der der Prozess laufen muss, der auf diesem Socket antwortet, bei jeder Verbindung vom Kernel gelesen (`SO_PEERCRED`): Ein Socket, den jemand anderes gebunden hat, bekommt nichts. Nicht gesetzt ist sie 0 |
| `allowed_uids` | optional, nur Unix-Socket: die einzigen uids, deren Aufrufe beantwortet werden; jede andere ergibt `PERMISSION_DENIED` und wird protokolliert. Es kann nur **einschränken**, was der Modus 0600 des Sockets erlaubt (sein Eigentümer und root) – root etwa darf dann nicht aufrufen. Über TCP abgelehnt, wo das Client-Zertifikat die Prüfung ist |

Der Pfad eines Volumes muss wie jede andere Quelle unter `source_roots` liegen.

`fileshare provisioner`, als root:

```hcl
# /etc/fileshare-provisioner.hcl
provisioner {
  listen     = "unix:///run/fileshare-provisioner/provisioner.sock"
  client_uid = 990
  group      = "fileshare"
  max_volume = "10T"
  state_file = "/var/lib/fileshare-provisioner/volumes.json"

  parent "tank" {
    zfs  = "tank/fileshare"
    root = "/srv/fileshare/volumes/tank"
  }
  parent "fast" { btrfs = "/srv/fileshare/volumes/fast" }
  parent "plain" {
    xfs         = "/srv/fileshare/volumes/plain"
    project_ids = "100000-199999"
  }
}
```

| Schlüssel | |
|---|---|
| `listen` | `unix://<absolute path>` und nichts anderes. Der Socket wird mit 0660 angelegt, im Besitz des Benutzers des Provisioners (root) und von `<group>`, und muss in einem Verzeichnis liegen, in das niemand außer dem Provisioner schreiben darf – seinem **eigenen** (`/run/fileshare-provisioner`), nicht dem von fileshare |
| `client_uid` | die einzige beantwortete uid: die von fileshare. **0 wird abgelehnt** |
| `group` | besitzt jede Volume-Wurzel (`root:<group>`, Modus 2770). fileshare muss darin Mitglied sein: Es schreibt über die Gruppe in ein Volume, nie als dessen Eigentümer – der Eigentümer eines Verzeichnisses kann dessen Projekt-ID ohne Privileg löschen und so einer XFS/ext4-Quota entkommen |
| `max_volume` | die größte Quota, die ein Volume haben darf (`"10T"`, `"500G"`, `"1048576"`); darüber gibt es `OUT_OF_RANGE` |
| `state_file` | was dieser Prozess angelegt hat – btrfs-Subvolume-IDs, XFS/ext4-Projekt-IDs –, damit er nach einem Neustart weiß, was ihm gehört |
| `parent "<id>"` | wo Volumes angelegt werden dürfen, unter der ID, die eine Anfrage nennt. Genau eines von `zfs` (das Dataset, dessen Kinder die Volumes sind, mit `root`, wo sie eingehängt werden), `btrfs`, `xfs` oder `ext4` (das Verzeichnis, in dem sie angelegt werden) |
| `project_ids` | XFS/ext4: der Bereich der Projekt-IDs, die dieser Parent vergeben darf |
| `enable_quota` | btrfs: erlaubt dem Provisioner, Quotas einzuschalten, wenn sie aus sind. Ohne das verhindert ein btrfs-Parent mit ausgeschalteten Quotas den Start – qgroups verlangsamen jeden Commit, wenn sich Snapshots vermehren, und das ist die Entscheidung des Betreibers |

Die Datei des Provisioners enthält nichts außer ihrem `provisioner`-Block: Die eigene Datei von fileshare,
ihm versehentlich übergeben, wird abgelehnt statt halb gelesen.

Ein Volume ist immer `<parent root>/<name>`: das Dataset `tank/fileshare/<name>`,
eingehängt unter `/srv/fileshare/volumes/tank/<name>` (mit `refquota`), ein
btrfs-Subvolume, begrenzt durch seine qgroup, ein XFS- oder ext4-Verzeichnis mit einer Projekt-ID
aus dem Bereich und einem harten Blocklimit. XFS und ext4 brauchen die Mount-Option `prjquota`
(ext4 zusätzlich die Features `project,quota`).

## Das systemd-Paar {#the-systemd-pair}

Der Provisioner:

```ini
# fileshare-provisioner.service
[Service]
ExecStart=/usr/local/bin/fileshare provisioner -c /etc/fileshare-provisioner.hcl
RuntimeDirectory=fileshare-provisioner
RuntimeDirectoryMode=0755
StateDirectory=fileshare-provisioner
StateDirectoryMode=0700
CapabilityBoundingSet=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
# Run as root, or as a dedicated user with:
# AmbientCapabilities=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
NoNewPrivileges=yes
SystemCallFilter=@system-service @mount quotactl quotactl_fd
SystemCallArchitectures=native
LockPersonality=yes
RestrictRealtime=yes
# NOT ProtectSystem= nor ReadWritePaths=: they give the service its own
# mount namespace, and the ZFS volumes it mounts would be seen by nobody
# else. And no Landlock: a Landlock-restricted thread cannot mount(2).
```

Der Server, ganz ohne Capability:

```ini
# fileshare.service
[Unit]
Requires=fileshare-provisioner.service
After=fileshare-provisioner.service

[Service]
User=fileshare
Group=fileshare
ExecStart=/usr/local/bin/fileshare serve -c /etc/fileshare/fileshare.hcl
RuntimeDirectory=fileshare
StateDirectory=fileshare
# No capability at all: CAP_SYS_RESOURCE would let it past an ext4 quota,
# and fileshare refuses to serve volumes with it.
CapabilityBoundingSet=
AmbientCapabilities=
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/srv/fileshare/volumes /var/lib/fileshare
```

`ProtectSystem=` ist für den Server in Ordnung: Es ist der Provisioner, der einhängt, und
Mounts propagieren in den Namespace des Servers hinein, nicht aus ihm heraus, sodass ein ZFS-Volume,
das nach dem Start des Servers eingehängt wurde, für ihn sichtbar ist.

{{< callout type="info" >}}
**Beispiele, keine getesteten Units**

Keine der beiden Units wird in der CI ausgeführt: Der End-to-End-Job startet beide Prozesse
unter `sudo`.
{{< /callout >}}

{{< callout type="warning" >}}
**Aktualisieren Sie zuerst den Provisioner**

Der Provisioner lehnt eine Nachricht ab, die ein Feld enthält, das seine Version nicht
definiert – er schließt die Verbindung –, also laufen `fileshare serve` und
`fileshare provisioner` in derselben Version.
{{< /callout >}}

## Die Volume-Aufrufe der Admin-API {#the-admin-apis-volume-calls}

`fileshare serve` leitet diese an den Provisioner weiter
([`admin.proto`](https://github.com/go-fileshare/fileshare/blob/v0.24.0/proto/fileshare/admin/v1/admin.proto)):

| | |
|---|---|
| `ListParents` | wo Volumes angelegt werden dürfen: ID, Art, Wurzel, freier und gesamter Platz jedes Parents, ob er Snapshots unterstützt; und `max_volume_bytes` |
| `CreateVolume` | ein Volume mit `quota_bytes` (0 wird abgelehnt). Idempotent nach Name: Dieselbe Quota antwortet mit dem vorhandenen Volume (`created` false), eine andere Quota ergibt `ALREADY_EXISTS` |
| `ResizeVolume` | die Quota ändern, in beide Richtungen – nie unter das Belegte (`FAILED_PRECONDITION`) |
| `SnapshotVolume` | ein schreibgeschützter, benannter Snapshot: ZFS und btrfs; `UNIMPLEMENTED` bei XFS und ext4 |
| `GetVolume`, `ListVolumes` | jedes Volume mit seiner Quota, seiner Belegung, seinen Snapshots und **den Freigaben, die es nutzen** |
| `DeleteVolume` | abgelehnt, solange eine Freigabe es nutzt – löschen Sie zuerst die Freigabe. Ein Volume mit Daten oder Snapshots braucht `destroy_data` |
| `CreateShare` mit `volume { parent, name }` | eine aus dem Volume bereitgestellte Freigabe, als Verzeichnis |

`DeleteShare` löscht nie ein Volume. „Die Freigaben, die es nutzen" sind die daraus
erstellten Freigaben **und** jede Freigabe – auch aus den Konfigurationsdateien, bereitgestellt,
deaktiviert oder nicht verfügbar –, deren Image oder Verzeichnis darin liegt.

Ein Name entspricht `^[a-z0-9][a-z0-9_-]{0,62}$`, und eine Anfrage trägt nie einen
Pfad: Sie nennt einen `parent` aus der eigenen Konfiguration des Provisioners.

Mit `reflection = true` im `admin`-Block kann `grpcurl` die Aufrufe ausführen.
Der Socket hat 0600, und `allowed_uids` oben nennt nur die uid von fileshare, also führen Sie
es als dieser Benutzer aus:

```sh
S=/run/fileshare/admin.sock
A=fileshare.admin.v1.AdminService

sudo -u fileshare grpcurl -plaintext -unix $S $A/ListParents

# a 32 GiB volume on the XFS parent
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "34359738368"}' \
  $A/CreateVolume

# a share served from it
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"name": "projects",
       "volume": {"parent": "plain", "name": "projects"},
       "grants": [{"subject": {"group": "staff"}, "access": "ACCESS_WRITE"}]}' \
  $A/CreateShare

# grow it to 64 GiB
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "68719476736"}' \
  $A/ResizeVolume
```

### Statuscodes {#status-codes}

Die Codes des Provisioners kommen so zurück, wie er sie gegeben hat: `INVALID_ARGUMENT` (ein Name
außerhalb der Grammatik, ein unbekannter Parent, eine Quota von 0), `OUT_OF_RANGE` (über
`max_volume`), `RESOURCE_EXHAUSTED` (die Quota oder der Zuwachs einer Größenänderung ist mehr,
als der Parent gerade frei hat; oder der Bereich der Projekt-IDs ist aufgebraucht),
`ALREADY_EXISTS`, `NOT_FOUND`, `FAILED_PRECONDITION`, `UNIMPLEMENTED`.

Drei sind fileshares eigene:

- kein `provisioner` konfiguriert: `FAILED_PRECONDITION`;
- der Provisioner lehnt die uid dieses Servers ab: ebenfalls `FAILED_PRECONDITION` – das ist
  ein Fehler im Deployment (seine `client_uid` ist falsch), nicht etwas, das der
  Admin-Aufrufer nicht darf;
- der Provisioner antwortet nicht: `UNAVAILABLE`.

## Was geprüft wird, bevor ein Volume bereitgestellt wird {#what-is-checked-before-a-volume-is-served}

fileshare fragt den Provisioner, wo das Volume liegt, und prüft dann selbst:

- der Pfad ist sauber, absolut und – aufgelöst – unter `source_roots`;
- es ist ein Verzeichnis;
- `statfs` meldet das Dateisystem der Art (`ZFS_SUPER_MAGIC`,
  `BTRFS_SUPER_MAGIC`, `XFS_SUPER_MAGIC`, `EXT4_SUPER_MAGIC`);
- **ZFS**: Es ist ein Einhängepunkt – sein Gerät ist nicht das seines Parents. Das Verzeichnis eines nicht
  eingehängten Datasets nähme Schreibzugriffe ohne Quota an. (Welches Dataset dort
  eingehängt ist, wird nicht geprüft: Die Antwort des Provisioners sagt es nicht.)
- **btrfs**: Es ist die Wurzel eines Subvolumes – ein einfaches Verzeichnis unter dem Parent hat
  keine qgroup;
- **XFS und ext4**: Es trägt eine Projekt-ID ungleich null, die seine neuen Dateien
  erben – ein Verzeichnis außerhalb jedes Projekts wird unbegrenzt bereitgestellt.

Die Zustandsdatei speichert den **Namen** des Volumes, nicht seinen Pfad, und jeder Start fragt
und prüft erneut. Ein Volume, das weg ist, eine Prüfung nicht besteht oder dessen Provisioner
nicht innerhalb von 10 s antwortet, lässt seine Freigabe **definiert und nicht bereitgestellt** – gemeldet
beim Start, von [`fileshare check`]({{< relref "/configuration/check.md" >}}) und im
Feld `unavailable` der Freigabe –, während der Rest des Servers startet.
`EnableShare` versucht es erneut.

{{< callout type="error" >}}
**`fileshare serve` weigert sich, Volumes als root oder mit `CAP_SYS_RESOURCE` bereitzustellen**

ext4 lässt beide über eine Projekt-Quota hinaus schreiben (fs/quota/dquot.c,
`ignore_hardlimit`): root schrieb in der CI von go-fsctl/projquota 16 MiB in ein 8-MiB-Projekt.
XFS hat keine solche Ausnahme, aber die Regel ist eine einzige
Regel für den Prozess, für jede Art: fileshare liest seine effektive uid und
`CapEff`/`CapPrm` in `/proc/self/status` (eine erlaubte Capability ist nur ein
`capset(2)` von einer effektiven entfernt), und ein Status, den es nicht lesen kann, zählt als
deren Besitz. Es verweigert das **Bereitstellen**, nicht die Volume-Aufrufe – Speicher
als root anzulegen ist harmlos, als root hineinzuschreiben nicht. `fileshare check` sagt,
was davon vorliegt. Deshalb lehnt der Provisioner `client_uid = 0` ab, und die
Unit des Servers oben hat ein leeres `CapabilityBoundingSet=`.
{{< /callout >}}

## Eine volle Freigabe {#a-full-share}

XFS meldet ein volles Projekt mit `ENOSPC`; ext4, btrfs und ZFS melden `EDQUOT`. Seit
v0.21.1 antwortet jedes Protokoll mit „voll":

| Protokoll | kein Platz (`ENOSPC`) | eine Quota (`EDQUOT`) |
|---|---|---|
| WebDAV | `507 Insufficient Storage` | `507 Insufficient Storage` |
| NFS | `NFS3ERR_NOSPC` (28) | `NFS3ERR_DQUOT` (69), wie RFC 1813 und Linux' nfsd |
| SFTP | `SSH_FX_FAILURE`, „no space left on device (the share is full)" – Version 3 hat keinen Code dafür | dasselbe |
| SMB | `STATUS_DISK_FULL` (0xC000007F) | `STATUS_DISK_FULL`, wie Samba es tut („Windows apps need this, not NT_STATUS_QUOTA_EXCEEDED") |
| S3 | schreibgeschützt bereitgestellt | schreibgeschützt bereitgestellt |

In v0.21.0 antwortete SMB mit `STATUS_ACCESS_DENIED`, und NFS beantwortete eine Quota mit
`NFS3ERR_NOSPC`.

Das gilt für jede [Verzeichnisfreigabe]({{< relref "/configuration/shares.md" >}}), nicht nur
für Volumes – siehe den [Upgrade-Hinweis]({{< relref "/status.md" >}}).

## Die Größe, die ein Client sieht {#the-size-a-client-sees}

Jedes Protokoll fragt den Server bei jeder Abfrage nach der Größe einer Freigabe:
NFS mit `FSSTAT`, SMB mit `FileFsFullSizeInformation` und WebDAV mit den
Quota-Eigenschaften aus RFC 4331.

- **ZFS, XFS und ext4**: Die Größe ist das, was `statfs` innerhalb des Volumes sagt;
  der Kernel antwortet dort mit der Quota (`refquota` bei ZFS, die Projekt-Quota bei
  XFS und ext4).
- **btrfs** (seit fileshare v0.23.0): `statfs` von btrfs ignoriert qgroups und meldet
  das ganze Dateisystem, deshalb nimmt der Server die Zahlen des Provisioners. Die
  Größe ist die Quota des Volumes, der freie Platz ist die Quota abzüglich der
  referenzierten Bytes der qgroup, oder der freie Platz des Dateisystems, wenn der
  kleiner ist. Der Server fragt den Provisioner im Hintergrund, höchstens alle
  5 Sekunden, solange Clients fragen, und antwortet bis dahin mit den letzten Zahlen.
  btrfs aktualisiert den Zähler einer qgroup beim Abschluss einer Transaktion
  (standardmäßig alle 30 Sekunden), daher kann der freie Platz, den ein Client sieht,
  den Schreibvorgängen um so viel hinterherhinken. Vor v0.23.0 zeigte ein btrfs-Volume
  das ganze Dateisystem; die Quota galt trotzdem.

Der belegte Platz kommt vom Provisioner (`used_bytes`): Die Belegung eines XFS- oder
ext4-Projekts zu lesen erfordert `CAP_SYS_ADMIN`, das fileshare nicht hat.

## Was der Provisioner zusichert {#what-the-provisioner-promises}

- **Eine uid, vom Kernel gelesen.** Der Peer wird beim Verbinden mit `SO_PEERCRED`
  identifiziert, und jeder außer `client_uid` – root eingeschlossen – erhält
  `PERMISSION_DENIED`, bevor ein Byte der Anfrage dekodiert wird.
- **Eine geschlossene Menge von Verben.** Eine Methode außerhalb seines Dienstes oder eine Nachricht, die
  sich nicht dekodieren lässt oder ein Feld enthält, das er nicht definiert, erhält keine Antwort: Die
  Verbindung wird geschlossen.
- **Er fasst nie an, was er nicht angelegt hat.** ZFS: die eigene
  `fileshare:volume`-Eigenschaft des Datasets – am Dataset gesetzt, da Benutzereigenschaften
  vererbt werden und eine geerbte Markierung nicht seine eigene ist. btrfs: Subvolume-ID und
  UUID, festgehalten in `state_file`. XFS/ext4: eine Projekt-ID innerhalb des Bereichs des Parents *und*
  festgehalten. Geprüft vor jeder Größenänderung, jedem Snapshot und jedem Löschen.
- **Löschen verweigert Daten und Snapshots**, sofern `destroy_data` nicht gesetzt ist, und wird
  protokolliert, bevor es geschieht.

## Gemessen {#measured}

In der CI von fileshare, als root auf echten Pools und Loop-Dateisystemen, mit `fileshare serve` als `nobody` und gesteuert über die
Admin-API: Schreibzugriffe über WebDAV und über SFTP enden bei allen vier Arten an der Quota eines 32-MiB-Volumes.

## Noch nicht {#not-yet}

- **Ceph.** CephFS-Quotas und RBD kommen in einer späteren Phase.
