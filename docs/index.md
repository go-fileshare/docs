# go-fileshare

**One disk image, served over SMB, NFS, WebDAV and SFTP — the same users, the
same per-share access, from one configuration file.** Pure Go,
`CGO_ENABLED=0`, one binary.

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
       block lifts this: sec=krb5 carries a principal a ticket proves
```

That last paragraph is the shape of the whole program: a rule the configuration
asks for and a protocol that cannot honour it do not quietly meet in the middle.
See [What a protocol can promise](protocols/index.md).

## Where to go next

| | |
|---|---|
| [The configuration](configuration/index.md) | users, groups, shares, `serve` blocks |
| [Shares, filesystems and partitions](configuration/shares.md) | what is detected, what must be declared |
| [Users, groups and directories](configuration/identity.md) | files, SQL, LDAP, and what each can prove |
| [`check`](configuration/check.md) | read the whole configuration back before restarting |
| [Protocols](protocols/index.md) | what each one can and cannot promise |
| [Building only what you want](operations/build-tags.md) | a tag leaves a protocol out entirely |
| [One process per protocol](operations/isolation.md) | `--isolate` |
| [Status](status.md) | what is verified, and what is not here yet |

## Licence

BSD-3-Clause.
