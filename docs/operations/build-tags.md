# Building only what you want

Each protocol is behind a build tag, and a tag leaves it out **entirely**: no
listener, no parser, no dependency, no code.

```sh
go install -tags nonfs,nowebdav,nosftp github.com/go-fileshare/fileshare@latest   # SMB only
go build   -tags nosmb,nonfs,nosftp .                                            # WebDAV only
```

Where the **people** come from is behind tags of its own: `nosql` leaves out
the three database drivers, `noldap` the LDAP client.

| build | size |
|---|---|
| everything | 28.6 MB |
| `-tags noldap` | 28.3 MB |
| `-tags nosftp` | 27.9 MB |
| `-tags nonfs,nowebdav,nosftp` (SMB only) | 26.4 MB |
| `-tags nopartitioned` (no apfs, btrfs, xfs, zfs) | 29.5 MB |
| `-tags nosql` | 16.9 MB |
| `-tags nosql,noldap` | 16.6 MB |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp` | 11.9 MB |

`nosql` is by a distance the biggest lever: PostgreSQL, MySQL and SQLite
together weigh **11.7 MB**, more than every protocol in this program put
together. A site whose users are in the file wants it. The drivers are imported
by the command and not by the library that uses them, which is what makes the
choice a build tag rather than a fork.

!!! note "A protocol this binary was built without"
    A configuration naming one is told *that*, rather than "there is no such
    protocol" — the difference between a typo and a build tag.

## Why not subprocess plugins

It was **measured rather than argued**: `hashicorp/go-plugin` brings gRPC and
protobuf, which cost **13.2 MB on their own** — more than this entire binary
with all three protocols and every driver in it. A plugin host would be twice
the size before loading anything, and each plugin binary would carry gRPC
again.

For attack surface, a tag is also the stronger tool for anything you do not
run: **code that was never compiled cannot be reached**, sandboxed or not.

What tags do *not* give is isolation between the protocols you **do** run, nor
a way to add a protocol without recompiling. Both are real, and both are
arguments for a process boundary rather than for a smaller binary — which is
[one process per protocol](isolation.md), using this same binary, not a plugin
framework.
