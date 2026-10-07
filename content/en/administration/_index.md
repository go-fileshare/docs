---
title: "The admin API"
linkTitle: "Administration"
weight: 30
description: "TODO"
tags: [administration, admin api, grpc]
---

Since v0.8.0 the shares can change while the server runs. An `admin` block
turns on a gRPC service that creates, changes and takes shares offline without
a restart and without editing a file.

```hcl
admin {
  listen       = "unix:///run/fileshare/admin.sock"   # made 0600
  state_file   = "/var/lib/fileshare/shares.json"
  source_roots = ["/srv/images", "/data"]
}
```

Nothing listens unless the block is written.

## What it does

The service is
[`fileshare.admin.v1.AdminService`](https://github.com/go-fileshare/fileshare/blob/v0.21.0/proto/fileshare/admin/v1/admin.proto):

| | |
|---|---|
| `CreateShare`, `UpdateShare`, `DeleteShare` | define a share from an image, a directory or (since v0.21.0) a volume; change `read_only` or `protocols` (its source cannot change: delete it and create another) |
| `DisableShare`, `EnableShare` | take a share offline and back |
| `Grant`, `Revoke` | give a subject read or write access, or take it away |
| `ListShares`, `GetShare` | every share, with what it serves, over which protocols, and why not the others |
| `ListUsers`, `ListGroups` | who a grant can name; each user with the protocols their credentials can answer |
| `ReloadDirectory` | read the people again, now — see [reading the directory again]({{< relref "/administration/reload.md" >}}) |
| `GetServerInfo` | the name, version, start time, generation and listeners |
| `ListParents`, `CreateVolume`, `ResizeVolume`, `SnapshotVolume`, `DeleteVolume`, `GetVolume`, `ListVolumes` | storage created through a privileged provisioner, and shares served from it — see [volumes]({{< relref "/administration/volumes.md" >}}) (since v0.21.0) |

A grant names a **user**, a **group** (`@group` in a configuration file), an
**`oidc:groups:` value** or an **`oidc:user:` name** — the vocabulary the
configuration file uses, see
[people the identity provider names]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).

`grpc.health.v1` answers on the same listener. `reflection = true` in the block
turns on gRPC server reflection, for `grpcurl`.

### Every change says what serving it did

A change answers with an `Applied` message:

```proto
message Applied {
  uint64 generation = 1;          // the generation now being served
  uint64 connections_closed = 2;  // how many the previous one had open, and closed
}
```

That is why every RPC has a response message of its own rather than returning
the share itself: the `.proto` follows [buf](https://buf.build)'s `STANDARD`
lint rules, checked by `buf lint` in the fileshare CI, over AIP-131's
"return the resource" — the wrapper is what carries `Applied`.

## Where it listens, and who may call it

{{< callout type="error" >}}
**Over TCP it is mutual TLS or nothing**

`tls_cert_file`, `tls_key_file` and `client_ca_file`, all three,
**loopback included**: any local user can reach loopback, and this API
decides who reads whose files. A unix socket is made **0600**, and its
permissions are its access control.
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

The listener is
[grpc-transports/control](https://github.com/grpc-transports/control). Each
change is logged with who made it: the client certificate's CN, or the socket
peer's uid.

## What it will and will not touch

**It manages the shares it created.** A share written in the configuration is
listed, with the same fields, and a change to its definition is refused with
`FAILED_PRECONDITION`: a share defined in two places is a question nobody wants
to answer, and the file is where that one is defined. The API's shares live in
`state_file`, written atomically, and are served again at the next start — which
is why a block without `state_file` is refused.

**A source must lie under `source_roots`**, resolved — links followed, `..`
taken out — and it is the resolved path that is kept. Without `source_roots` the
API **cannot create shares at all**: the process can read `/dev` and `/etc`,
and nobody meant to hand those to a caller. It is checked again at every start,
and a share in `state_file` that is no longer under a root stops the start,
naming it.

**Every share has at least one grant.** A share with none is open to anyone who
authenticates. The file may say that on purpose; an API call should not say it
by omission. So `CreateShare` needs a grant, and revoking the last one is
refused — delete the share instead.

**Names a protocol can carry.** A share name is at most 80 characters, does
not begin or end with a space, is not `.` or `..`, and holds no unprintable
character and none of `:*?"<>|{}%`. Subjects hold no unprintable character
either, and one share takes at most 1000 grants — a group's job long before.
Each of these was accepted before v0.14.0, written to `state_file`, and then
fatal at every start.

**No share may contain** the configuration, the state file, or the secrets
they name: whoever writes into it would rewrite who may do what. Since v0.17.0
that includes `authorized_keys_file`, a `users` block's `dsn_file` and
`bind_password_file`, the `ssf` block's `ca_file`, and a sqlite database a DSN
names: a writer who could add their key to somebody's `authorized_keys` would
log in as them. **Nor may a share contain another share** (see
[directories]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})).

**A change is checked like a configuration, opened, written down, then
served.** A change the server cannot honour — an image that will not open, a
name nobody in the directory has — is refused, with what was served and what
was written left as they were.

## Disabling a share

**Disabling is Samba's `available = no`**: the share stays defined and every
attempt to connect fails; its open connections are closed and its image or
directory is **let go of**, so the file can be replaced while it is offline.
`EnableShare` serves it again, and is refused when its source can no longer be
opened. `CreateShare` can also create a share already disabled.

Unlike every other change it applies to a share **of the configuration** too —
taking a share offline is an operation, not a definition — and it survives a
restart: it is kept in the state file. [`fileshare check`]({{< relref "/configuration/check.md" >}})
lists what is offline.

## A change is a new generation

{{< callout type="error" >}}
**A change restarts the protocol servers, and closes their connections**

SMB checks who may connect once per tree connect, and SFTP builds a
person's tree once per login, so a session that outlived a revocation would
keep the access just taken away. "Revoked, except for whoever was already
connected" is not a revocation.
{{< /callout >}}

So a change is not applied *to* the running servers: they are replaced by new
ones built from the new list — a **generation** — the way a restart would,
without giving up the ports. The libraries could not do otherwise:
go-filesystems/smb has `Share` and no `Unshare`, nfs has `Export` and no
`Unexport`.

The ports stay bound, so a client connecting during a change **waits** instead
of being refused. Images whose share did not change keep their driver, never
opened a second time. Clients reconnect; that is the price, paid for every
change — which is why the API applies one change per call rather than one per
field.

## Limits

- `--isolate` does not go with an `admin` block — nor a `metrics` block — yet:
  there is no one process a change could be applied to. The combination is
  refused.
- A binary built with `-tags nogrpc` has no admin API, and a configuration with
  an `admin` block is **refused** rather than served without one. That tag saves
  11.7 MB; see [building only what you want]({{< relref "/operations/build-tags.md#the-admin-api-and-nogrpc" >}}).

## Changing the `.proto`

The Go code under `proto/` is generated and committed, so `go install` needs no
`protoc`. After changing the `.proto`, regenerate it with the versions the
fileshare CI pins (protoc 34.1, protoc-gen-go v1.36.12, protoc-gen-go-grpc
v1.6.2) — CI regenerates it and fails when the committed code differs:

```sh
protoc -I proto --go_out=. --go_opt=module=github.com/go-fileshare/fileshare \
  --go-grpc_out=. --go-grpc_opt=module=github.com/go-fileshare/fileshare \
  proto/fileshare/admin/v1/admin.proto
```
