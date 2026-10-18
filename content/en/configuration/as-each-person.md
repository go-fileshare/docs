---
title: "Directories served as each person"
weight: 15
description: "identity = \"system\": a directory share whose files are opened by a worker running as the person who logged in, so the kernel applies the tree's permissions and what they create belongs to them."
tags: [configuration, shares, identity, security]
---

Ordinarily one fileshare process serves everybody as one account: fileshare
decides who may do what, and every file it creates belongs to that account.
A directory share can instead be served **as each person**. Whoever logged
in has their files opened by a process running with their uid, gid and
groups, so the kernel applies the tree's permissions and what they create
belongs to them.

```hcl
identity {
  service_user = "fileshare"   # the account the protocols are served as
  min_uid      = 1000          # the default; uid 0 is refused whatever this says
  min_gid      = 1000
  deny_uids    = [65534]
  idle         = "2m"          # how long an unused worker is kept
}

share "home" {
  directory = "/srv/home"
  identity  = "system"
}
```

## Three processes {#three-processes}

`fileshare serve` must be **started as root**. It stays root only to start
each person's worker. Everything that faces the network runs in a second
process, as `service_user`.

| process | runs as | does |
|---|---|---|
| the **launcher** (`fileshare serve`) | root | binds the listeners, starts the front, starts workers when the front asks. It speaks no protocol. |
| the **front** | `service_user`, with that account's groups | speaks the protocols and authenticates, as `serve` always has. For such a share it asks the launcher for a **person and a share**, never a uid or a path. |
| a **worker** | the person | serves one share to the front, over a socket the launcher handed across |

A worker first checks that it is running as exactly the expected uid, gid
and groups, and that `setuid(0)` fails. Only then does it open the tree,
through an `os.Root`. One worker serves all of that person's connections to
the share, over SFTP and WebDAV alike, and it stops once nothing has used it
for `idle`. When the server stops, the workers stop with it.

## Who a person is on this system {#who-a-person-is-on-this-system}

The launcher resolves the person **itself**, in this order:

1. **The directory**, if it carries `uidNumber` and `gidNumber`:
   - a `user` block's `attributes = { uidNumber = "2001", gidNumber = "2001" }`;
   - an LDAP source with `attributes = ["uidNumber", "gidNumber"]`;
   - an SQL source's `users_attributes` query.

   The supplementary groups are the groups that list the person (a `group`
   block's `members`, `memberUid`) and those the person names (`memberOf`),
   each by its `gidNumber`. A group without one is not a system group.
2. **`/etc/passwd` and `/etc/group`**, by name.

**Refused**: uid 0, gid 0, `(uid_t)-1`, a uid below `min_uid`, a primary gid
below `min_gid`, and every uid in `deny_uids`. A supplementary group below
`min_gid` is left out, and the log says so. A person found in neither place
is refused too: there is no shared fallback account. A person the identity
provider vouches for gets a worker only with
[`local_names`]({{< relref "/configuration/identity.md" >}}), which makes
their name a local one.

So a front that has been taken over can ask only for a person who exists
and is allowed, on a share that exists. It still decides **who** has
authenticated, and the launcher cannot check that for it.

## What a client sees {#what-a-client-sees}

| | SFTP | WebDAV |
|---|---|---|
| a worker is taken | at login, for the whole connection | per request; locks live as long as the worker does |
| the launcher refuses the person | the share is not in their tree (a tree with nothing in it is a refused login) | `403` |
| no worker can be started | the same | `503` |

The reason goes to the server's log, never to the client.

{{< callout type="warning" >}}
**Not over SMB, NFS or S3 yet**

They refuse such a share, and [`check`]({{< relref "/configuration/check.md" >}})
says so. SMB and NFS need a filesystem per session in their libraries. NFS
will only ever do it under Kerberos, because `AUTH_UNIX` is the client's own
claim (RFC 5531 §14).
{{< /callout >}}

## What it asks of the system {#what-it-asks-of-the-system}

- **Unix**: Linux, the BSDs and macOS. On Windows the share is refused.
- **Readable by `service_user`**: the configuration and every file it names
  (password files, host key, certificates), since the front reads them.
- **Executable by the people**: the binary, since each worker runs it as
  its person.
- **A restart** to add or change such a share. The launcher reads the
  configuration once, and refuses a share it does not know rather than let
  the front serve it.
- `--isolate` does not go with it.

The design, and why each choice was made, is in
[docs/identity.md](https://github.com/go-fileshare/fileshare/blob/main/docs/identity.md).
