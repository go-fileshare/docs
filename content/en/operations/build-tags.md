---
title: "Building only what you want"
weight: 10
description: "TODO"
tags: [operations, build]
---

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
| everything | 31.2 MB |
| `-tags noldap` | 30.9 MB |
| `-tags nosftp` | 30.6 MB |
| `-tags noopenpubkey` (no opkssh logins over SFTP) | about 2 MB less |
| `-tags nos3` | 31.1 MB |
| `-tags nonfs,nowebdav,nosftp,nos3` (SMB only) | 28.1 MB |
| `-tags nopartitioned` (no apfs, btrfs, xfs, zfs) | 29.2 MB |
| `-tags nosql` | 20.2 MB |
| `-tags nosql,noldap` | 19.8 MB |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp,nos3` | 16.2 MB |

These are the figures the fileshare README gives, measured before the admin
API arrived; the table below is the later measurement.

## The admin API, and `nogrpc`

The [admin API]({{< relref "/administration/_index.md" >}}) brought gRPC and protobuf in, and
they are behind `nogrpc` (linux/amd64, measured 2026-09-29, when the rows above
had grown to 34.1 MB for everything):

| build | size |
|---|---|
| everything, with the admin API | 46.1 MB |
| `-tags nogrpc` | 34.4 MB |
| `-tags nosql,noldap,nogrpc` | 22.5 MB |

gRPC costs **11.7 MB** — the [plugin measurement](#why-not-subprocess-plugins)
below, taken again, now that it is here for a reason of its own. A site that
manages its shares in files does not need to carry it.

{{< callout type="warning" >}}
**An `admin` block in a `nogrpc` build is refused**

Rather than served without one: a configuration that asks for the API and
starts without it does not do what it says.
{{< /callout >}}

`nogrpc` leaves out the admin API only. [Health and metrics]({{< relref "/administration/health.md" >}})
are plain HTTP and stay.

`nosql` is by a distance the biggest lever: PostgreSQL, MySQL and SQLite
together weigh **11.7 MB**, more than every protocol in this program put
together. A site whose users are in the file wants it. The drivers are imported
by the command and not by the library that uses them, which is what makes the
choice a build tag rather than a fork.

{{< callout type="info" >}}
**A protocol this binary was built without**

A configuration naming one is told *that*, rather than "there is no such
protocol" — the difference between a typo and a build tag. The same holds
for `opkssh_client_id` under `noopenpubkey` and an `admin` block under
`nogrpc`.
{{< /callout >}}

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
[one process per protocol]({{< relref "/operations/isolation.md" >}}), using this same binary, not a plugin
framework.
