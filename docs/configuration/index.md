# The configuration

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

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" { addr = "0.0.0.0:8080" }
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

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
| `totp_secret` | a base32 one-time-code secret | a second factor, where a protocol has room for one |

⛔ `password` and `password_file` together are refused at startup, naming the
person: two answers to "what is their password" is a question about which one
wins, and a configuration should not have to be read twice to find out.

!!! tip "An inline user CAN be served over SMB"
    Until recently this block read only the password fields, so serving SMB
    to somebody written down here meant giving this file their password in
    the clear — or moving them into a database. With `nt_hash` it does not:
    a site that stores what Samba stores can write that instead. See
    [what a source can prove](identity.md#what-a-source-can-prove-and-what-each-protocol-needs),
    which is the same table one layer up.

A group is written `@name` wherever a person could be.

## What is refused at startup, rather than later

!!! warning "A name that belongs to no `user` block"
    `allow = ["alise"]` would otherwise lock Alice out of her own share and
    start happily. It is refused.

!!! warning "A share that names who may use it, over NFS"
    A configuration saying *photos belongs to alice* and a protocol handing
    photos to whoever connects cannot both be honoured. See
    [NFS](../protocols/nfs.md).

!!! warning "A provider with nowhere to put a token"
    An [`oidc`](../protocols/webdav.md) block without a WebDAV `serve` block is
    refused: no other protocol here can carry an `Authorization` header.

!!! warning "A protocol this binary was built without"
    It is told *that*, rather than "there is no such protocol" — the difference
    between a typo and a [build tag](../operations/build-tags.md).

A `serve` block with no `addr` lands on the registered port for that protocol,
on loopback.

## `protocols` on a share

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

Says which protocols carry it. Useful on its own — a share declared SMB-only is
a share the WebDAV process is never told about — and required in some
configurations under [`--isolate`](../operations/isolation.md). A `serve` block
that would end up carrying nothing is refused too, before any image is opened,
naming the shares that were kept from it and why.
