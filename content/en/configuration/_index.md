---
title: "The configuration"
linkTitle: "Configuration"
weight: 10
description: "How the HCL configuration is laid out, what a user block carries, and what is refused at startup."
tags: [configuration]
---

One file, or a directory of small ones. `--config /etc/fileshare.d` merges
them, so a user in one file and a share in another belong together.

```hcl
name = "ATTIC"

user "alice" { password_file = "/etc/fileshare/alice.pw" }
user "bob"   { password_file = "/etc/fileshare/bob.pw" }

group "family" { members = ["alice", "bob"] }

share "photos" {
  image   = "/srv/photos.img"
  allow   = ["@family"]        # a group, or a person, in either list
  writers = ["alice"]          # bob gets it read-only
}

share "scratch" {
  image = "/srv/scratch.img"   # anyone who authenticates, read-write
}

tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true                  # HTTP Basic is the password: never in the clear
}
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

{{< callout type="error" >}}
**Since v0.9.0, WebDAV with passwords is not served in the clear**

`serve "webdav" { addr = "0.0.0.0:8080" }` — the example this page showed
until then — is now **refused at startup** whenever somebody authenticates:
HTTP Basic is the password on every request. Say `tls = true` with a `tls`
block, as above, or `plaintext = true` when a proxy terminates TLS in front
of it. See [TLS]({{< relref "/security/tls.md" >}}).
{{< /callout >}}

A share can also be a [device]({{< relref "/configuration/shares.md#a-device-not-only-an-image" >}}) or a
[directory of the host]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}) instead of an
image.

## What a `user` block can carry

Every example above uses `password_file`, which is the common case and was
for a while the only one this program read. It is not the whole block, and
the difference decides **which protocols a person can be served over** — so
it is worth having in one place rather than discovering at a mount.

| field | what it is | what it buys |
|---|---|---|
| `password` | a password, in the file | every protocol a password answers |
| `password_file` | the same, in a file of its own | the same, without the secret in the configuration |
| `nt_hash` | `MD4(UTF16LE(password))`, 32 hex characters | **SMB**, for a person whose password this site does not hold |
| `authorized_keys` | `authorized_keys` lines, inline | **SFTP** |
| `authorized_keys_file` | the same, from a file | **SFTP** |
| `totp_secret` | a base32 one-time-code secret | nothing yet: it is read and checked, and no protocol here asks for a second factor |

⛔ `password` and `password_file` together are refused at startup, naming the
person: two answers to "what is their password" is a question about which one
wins, and a configuration should not have to be read twice to find out. So are
`authorized_keys` and `authorized_keys_file` together.

⛔ A `user` block with none of `password`, `password_file`, `authorized_keys`
or `authorized_keys_file` is refused too — *has no way to authenticate* —
unless `trusted_user_ca_file` is set. `nt_hash` and `totp_secret` do not count
here.

{{< callout type="default" >}}
**An inline user CAN be served over SMB**

Until recently this block read only the password fields, so serving SMB
to somebody written down here meant giving this file their password in
the clear — or moving them into a database. With `nt_hash` it does not:
a site that stores what Samba stores can write that instead. The block still
needs one of the fields above beside it — keys, say — or
`trusted_user_ca_file`: `nt_hash` alone is refused at startup. See
[what a source can prove]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}),
which is the same table one layer up.
{{< /callout >}}

A group is written `@name` wherever a person could be.

## What is refused at startup, rather than later

{{< callout type="warning" >}}
**A name that belongs to nobody**

`allow = ["alise"]` would otherwise lock Alice out of her own share and
start happily. A name no `user` block and no [`users` directory]({{< relref "/configuration/identity.md" >}})
knows is refused, and so is a `@group` that none of them has.
{{< /callout >}}

{{< callout type="warning" >}}
**A share that names who may use it, over NFS**

A configuration saying *photos belongs to alice* and a protocol handing
photos to whoever connects cannot both be honoured. Such a share is not
exported over NFS — unless a `kerberos` block or `identity = "certificate"`
lets NFS tell people apart — and an `nfs` serve block left with nothing to
carry is refused. See [NFS]({{< relref "/protocols/nfs.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**A provider with nowhere to put its word**

An [`oidc`]({{< relref "/protocols/webdav.md" >}}) block is refused unless WebDAV is
served — the one protocol that carries an `Authorization` header — or SFTP
is served with the provider's certificates turned on (`ssh_ca_file` or
`opkssh_client_id`, see [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})).
SMB and NFS have nowhere to put either.
{{< /callout >}}

{{< callout type="warning" >}}
**Passwords in the clear**

WebDAV on an address other machines can reach, without `tls = true` or
`plaintext = true`, while anybody authenticates. See
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**A block that cannot be served safely**

An `admin` block without `state_file`, or listening on TCP without mutual
TLS; a `tls` block that no `serve` block uses; a `reload` shorter than a
second; a revocation list fetched over plain HTTP. Each is described
where it belongs: [administration]({{< relref "/administration/_index.md" >}}),
[TLS]({{< relref "/security/tls.md" >}}), [reload]({{< relref "/administration/reload.md" >}}),
[revocation]({{< relref "/security/_index.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**A protocol this binary was built without**

It is told *that*, rather than "there is no such protocol" — the difference
between a typo and a [build tag]({{< relref "/operations/build-tags.md" >}}).
{{< /callout >}}

A `serve` block with no `addr` listens on loopback, `127.0.0.1`, on a port
that needs no privilege: 4445 for SMB, 8080 for WebDAV, 2222 for SFTP, 2049
for NFS, 9000 for S3.

## `protocols` on a share

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

Says which protocols carry it. Useful on its own — a share declared SMB-only is
a share the WebDAV process is never told about — and required in some
configurations under [`--isolate`]({{< relref "/operations/isolation.md" >}}). A `serve` block
that would end up carrying nothing is refused too, before any image is opened,
naming the shares that were kept from it and why — unless an `admin` block is
there to give it shares later.
