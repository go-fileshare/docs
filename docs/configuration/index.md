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
```

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
