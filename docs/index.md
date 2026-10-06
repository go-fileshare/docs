# go-fileshare

**A disk image — or a directory — served over SMB, NFS, WebDAV, SFTP and S3 —
the same users, the same per-share access, from one configuration file.** Pure
Go, `CGO_ENABLED=0`, one binary. These pages describe
[v0.21.1](status.md).

```sh
go install github.com/go-fileshare/fileshare@latest

fileshare --image disk.img --user alice --password-file pw   # one image, now
fileshare --config /etc/fileshare.d                          # several, with users
```

The password comes from a **file**, never a flag: an argument is visible in the
process list to every user on the machine.

## Why one program

Which protocol carries an image is a property of the client at the other end:
macOS and Windows reach for SMB, a Linux fleet already has NFS, a browser or a
phone has HTTP. Running three servers, each with its own configuration file and
its own idea of who `alice` is, is a way to get two of them subtly wrong.

So there is one configuration, one set of users, one set of per-share rules —
and the protocols are listeners over it.

## What it prints at startup

```
smb    on 0.0.0.0:445  — photos and scratch
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

That last paragraph is the shape of the whole program: a rule the configuration
asks for and a protocol that cannot honour it do not quietly meet in the middle.
See [What a protocol can promise](protocols/index.md).

## Where to go next

| | |
|---|---|
| [The configuration](configuration/index.md) | users, groups, shares, `serve` blocks |
| [Shares: images, devices, directories](configuration/shares.md) | what is detected, what must be declared |
| [Users, groups and directories](configuration/identity.md) | files, SQL, LDAP, and what each can prove |
| [`check`](configuration/check.md) | read the whole configuration back before restarting |
| [Protocols](protocols/index.md) | what each one can and cannot promise |
| [The admin API](administration/index.md) | shares created, granted and taken offline without a restart |
| [Volumes](administration/volumes.md) | ZFS datasets, btrfs subvolumes and XFS/ext4 project quotas, created through the API and served |
| [Health and metrics](administration/health.md) | `/healthz`, `/readyz`, `/metrics` |
| [Reading the directory again](administration/reload.md) | `reload`, `SIGHUP`, and what a removal does to open sessions |
| [TLS](security/tls.md) | files or ACME; and why WebDAV is refused in the clear |
| [What revokes what](security/index.md) | KRL, CRL, shared signals — and why each fails closed |
| [Building only what you want](operations/build-tags.md) | a tag leaves a protocol out entirely |
| [One process per protocol](operations/isolation.md) | `--isolate` |
| [Status](status.md) | what is verified, and what is not here yet |

## Licence

BSD-3-Clause.
