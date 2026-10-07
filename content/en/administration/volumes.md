---
title: "Volumes: storage fileshare creates"
weight: 10
description: "Creating ZFS, btrfs, XFS and ext4 storage through the privileged fileshare provisioner, and serving shares from it."
tags: [administration, volumes, zfs, btrfs, quotas]
---

Since v0.21.0 the [admin API]({{< relref "/administration/_index.md" >}}) can **create the storage a share
serves** — a **ZFS dataset**, a **btrfs subvolume**, an **XFS or ext4 directory
under a project quota** — sized, owned and tracked, and serve a share from it.
Linux only. Ceph is not supported yet.

Creating that storage needs `CAP_SYS_ADMIN` (quotactl, the btrfs quota ioctls,
`/dev/zfs`, mount(2)). `fileshare serve` parses five network protocols for
strangers and holds no privilege, so the part that does is a second role of the
same binary, `fileshare provisioner`, behind a unix socket and a small, closed
protocol:

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

One binary rather than two daemons: one version to deploy, one protocol
definition. The storage itself is done by [go-fsctl](https://go-fsctl.github.io/)
(pure Go, no `zfs` or `btrfs` command). The design and what changed while
building it are in the fileshare repository's
[`docs/volumes.md`](https://github.com/go-fileshare/fileshare/blob/v0.22.2/docs/volumes.md).

## Both processes, configured

`fileshare serve`, as the user `fileshare` (uid 990):

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

| key | |
|---|---|
| `provisioner` | the provisioner's socket, `unix:///path`. Without it every volume call answers `FAILED_PRECONDITION` and no share is made from a volume |
| `provisioner_uid` | the uid the process answering on that socket must run as, read from the kernel (`SO_PEERCRED`) at every connection: a socket somebody else bound gets nothing. Unset, it is 0 |
| `allowed_uids` | optional, unix socket only: the only uids whose calls are answered; any other is `PERMISSION_DENIED`, and audited. It can only **narrow** what the socket's 0600 mode allows (its owner and root) — root, say, may then not call. Refused over TCP, where the client certificate is the check |

A volume's path must lie under `source_roots`, like any other source.

`fileshare provisioner`, as root:

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

| key | |
|---|---|
| `listen` | `unix://<absolute path>` and nothing else. The socket is made 0660, owned by the provisioner's user (root) and `<group>`, and must be in a directory nobody but the provisioner may write — its **own** (`/run/fileshare-provisioner`), not fileshare's |
| `client_uid` | the one uid answered: fileshare's. **0 is refused** |
| `group` | owns every volume root (`root:<group>`, mode 2770). fileshare must be in it: it writes into a volume through the group, never as its owner — the owner of a directory may clear its project id without privilege, and so step out of an XFS/ext4 quota |
| `max_volume` | the largest quota one volume may have (`"10T"`, `"500G"`, `"1048576"`); above it is `OUT_OF_RANGE` |
| `state_file` | what this process created — btrfs subvolume ids, XFS/ext4 project ids — so that after a restart it knows what is its own |
| `parent "<id>"` | where volumes may be created, under the id a request names. Exactly one of `zfs` (the dataset volumes are children of, with `root` where they are mounted), `btrfs`, `xfs` or `ext4` (the directory they are created in) |
| `project_ids` | XFS/ext4: the range of project ids this parent may give out |
| `enable_quota` | btrfs: let the provisioner turn quotas on when they are off. Without it a btrfs parent with quotas off stops the start — qgroups slow every commit as snapshots multiply, and that is the operator's choice |

The provisioner's file holds nothing but its `provisioner` block: fileshare's
own file handed to it by mistake is refused rather than half-read.

A volume is always `<parent root>/<name>`: dataset `tank/fileshare/<name>`
mounted at `/srv/fileshare/volumes/tank/<name>` (with `refquota`), a btrfs
subvolume limited by its qgroup, an XFS or ext4 directory with a project id
from the range and a hard block limit. XFS and ext4 need the `prjquota` mount
option (ext4 also the `project,quota` features).

## The systemd pair

The provisioner:

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

The server, with no capability at all:

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

`ProtectSystem=` is fine for the server: it is the provisioner that mounts, and
mounts propagate into the server's namespace, not out of it, so a ZFS volume
mounted after the server started is visible to it.

{{< callout type="info" >}}
**Examples, not tested units**

Neither unit is exercised in CI: the end-to-end job runs both processes
under `sudo`.
{{< /callout >}}

{{< callout type="warning" >}}
**Upgrade the provisioner first**

The provisioner refuses a message carrying a field its version does not
define — it closes the connection — so `fileshare serve` and
`fileshare provisioner` run the same version.
{{< /callout >}}

## The admin API's volume calls

`fileshare serve` relays these to the provisioner
([`admin.proto`](https://github.com/go-fileshare/fileshare/blob/v0.22.2/proto/fileshare/admin/v1/admin.proto)):

| | |
|---|---|
| `ListParents` | where volumes may be created: each parent's id, kind, root, free and total space, whether it takes snapshots; and `max_volume_bytes` |
| `CreateVolume` | a volume of `quota_bytes` (0 is refused). Idempotent by name: the same quota answers with the volume that exists (`created` false), another quota is `ALREADY_EXISTS` |
| `ResizeVolume` | change the quota, either way — never below what is used (`FAILED_PRECONDITION`) |
| `SnapshotVolume` | a read-only, named snapshot: ZFS and btrfs; `UNIMPLEMENTED` on XFS and ext4 |
| `GetVolume`, `ListVolumes` | each volume with its quota, what it uses, its snapshots and **the shares that use it** |
| `DeleteVolume` | refused while a share uses it — delete the share first. A volume holding data or snapshots needs `destroy_data` |
| `CreateShare` with `volume { parent, name }` | a share served from the volume, as a directory |

`DeleteShare` never deletes a volume. "The shares that use it" are the shares
made from it **and** any share — of the configuration files too, served,
disabled or unavailable — whose image or directory lies inside it.

A name matches `^[a-z0-9][a-z0-9_-]{0,62}$`, and a request never carries a
path: it names a `parent` from the provisioner's own configuration.

With `reflection = true` in the `admin` block, `grpcurl` can make the calls.
The socket is 0600 and `allowed_uids` above names only fileshare's uid, so run
it as that user:

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

### Status codes

The provisioner's codes come back as it gave them: `INVALID_ARGUMENT` (a name
outside the grammar, an unknown parent, a quota of 0), `OUT_OF_RANGE` (above
`max_volume`), `RESOURCE_EXHAUSTED` (the quota, or a resize's growth, is more
than the parent has free now; or the project-id range is used up),
`ALREADY_EXISTS`, `NOT_FOUND`, `FAILED_PRECONDITION`, `UNIMPLEMENTED`.

Three are fileshare's own:

- no `provisioner` configured: `FAILED_PRECONDITION`;
- the provisioner refuses this server's uid: `FAILED_PRECONDITION` too — it is
  a deployment mistake (its `client_uid` is wrong), not something the admin
  caller may not do;
- the provisioner does not answer: `UNAVAILABLE`.

## What is checked before a volume is served

fileshare asks the provisioner where the volume is, and then checks, itself:

- the path is clean, absolute and — resolved — under `source_roots`;
- it is a directory;
- `statfs` reports the kind's filesystem (`ZFS_SUPER_MAGIC`,
  `BTRFS_SUPER_MAGIC`, `XFS_SUPER_MAGIC`, `EXT4_SUPER_MAGIC`);
- **ZFS**: it is a mount point — its device is not its parent's. An unmounted
  dataset's directory would take writes under no quota. (Which dataset is
  mounted there is not verified: the provisioner's answer does not say.)
- **btrfs**: it is a subvolume's root — a plain directory under the parent has
  no qgroup;
- **XFS and ext4**: it carries a non-zero project id that its new files
  inherit — a directory outside every project is served unbounded.

The state file keeps the volume's **name**, not its path, and every start asks
and checks again. A volume that is gone, fails a check, or whose provisioner
does not answer within 10 s leaves its share **defined and not served** — said
at the start, by [`fileshare check`]({{< relref "/configuration/check.md" >}}), and in the
share's `unavailable` field — while the rest of the server starts.
`EnableShare` tries it again.

{{< callout type="error" >}}
**`fileshare serve` refuses to serve volumes as root or with `CAP_SYS_RESOURCE`**

ext4 lets either write past a project quota (fs/quota/dquot.c,
`ignore_hardlimit`): root wrote 16 MiB into an 8 MiB project in
go-fsctl/projquota's CI. XFS has no such exemption, but the rule is one
rule for the process, on every kind: fileshare reads its effective uid and
`CapEff`/`CapPrm` in `/proc/self/status` (a permitted capability is one
`capset(2)` away from effective), and a status it cannot read counts as
holding it. It refuses **serving**, not the volume calls — creating storage
as root is harmless, writing into it as root is not. `fileshare check` says
which it is. Hence `client_uid = 0` is refused by the provisioner, and the
server's unit above has an empty `CapabilityBoundingSet=`.
{{< /callout >}}

## A full share

XFS says a full project with `ENOSPC`; ext4, btrfs and ZFS say `EDQUOT`. Since
v0.21.1 every protocol answers "full":

| protocol | no space (`ENOSPC`) | a quota (`EDQUOT`) |
|---|---|---|
| WebDAV | `507 Insufficient Storage` | `507 Insufficient Storage` |
| NFS | `NFS3ERR_NOSPC` (28) | `NFS3ERR_DQUOT` (69), as RFC 1813 and Linux's nfsd |
| SFTP | `SSH_FX_FAILURE`, "no space left on device (the share is full)" — version 3 has no code for it | the same |
| SMB | `STATUS_DISK_FULL` (0xC000007F) | `STATUS_DISK_FULL`, as Samba does ("Windows apps need this, not NT_STATUS_QUOTA_EXCEEDED") |
| S3 | served read-only | served read-only |

In v0.21.0, SMB answered `STATUS_ACCESS_DENIED` and NFS answered a quota with
`NFS3ERR_NOSPC`.

This applies to every [directory share]({{< relref "/configuration/shares.md" >}}), not only
to volumes — see the [upgrade note]({{< relref "/status.md" >}}).

## The size a client sees

A share's size is what `statfs` says inside the volume: the quota for ZFS
(`refquota`), XFS and ext4 (project `statfs`). **For btrfs it is the whole
filesystem**: btrfs's `statfs` ignores qgroups, so `df` on a btrfs volume shows
the filesystem's size and free space, not the volume's quota — which still
holds.

Used space comes from the provisioner (`used_bytes`): reading an XFS or ext4
project's usage needs `CAP_SYS_ADMIN`, which fileshare does not have.

## What the provisioner promises

- **One uid, read from the kernel.** The peer is identified with `SO_PEERCRED`
  when it connects, and anybody but `client_uid` — root included — is
  `PERMISSION_DENIED` before a byte of the request is decoded.
- **A closed set of verbs.** A method outside its service, or a message that
  does not decode or carries a field it does not define, gets no answer: the
  connection is closed.
- **It never touches what it did not create.** ZFS: the dataset's own
  `fileshare:volume` property — set on the dataset, since user properties are
  inherited and an inherited mark is not its own. btrfs: the subvolume id and
  uuid recorded in `state_file`. XFS/ext4: a project id inside the parent's
  range *and* recorded. Checked before every resize, snapshot and delete.
- **Delete refuses data and snapshots** unless `destroy_data` is set, and is
  logged before it happens.

## Measured

In fileshare's CI, as root on real pools and loop-backed filesystems, with `fileshare serve` running as `nobody` and driven through the
admin API: writes over WebDAV and over SFTP stop at a 32 MiB volume's quota on
all four kinds.

## Not yet

- **Ceph.** CephFS quotas and RBD are a later phase.
