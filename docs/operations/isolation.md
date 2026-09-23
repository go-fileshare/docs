# One process per protocol

```sh
fileshare --config /etc/fileshare.d --isolate
```

```
smb    on 0.0.0.0:445  — process 19810, serving public, photos and scratch
webdav on 0.0.0.0:8080 — process 19811, serving public and photos
nfs    on 0.0.0.0:2049 — process 19812, serving public
```

The parent binds the listeners and then execs **itself** once per protocol,
handing each child only the shares that protocol may serve.

## Checked with `lsof`, not by reading the code

```
smb     (pid 19810) has open: scratch.img photos.img
webdav  (pid 19811) has open: photos.img
nfs     (pid 19812) has open: photos.img
```

The writable image is open in **exactly one process**, and the WebDAV child
never opens it at all.

That is what [build tags](build-tags.md) cannot give: a panic or an exhausted
heap in one protocol takes down one protocol, each child can be confined by
whatever the operating system offers, and a bug in one parser cannot reach an
image that process never opened.

## Privileged ports

On Unix the parent passes the bound listener to the child, so **a privileged
port works with unprivileged children** — the parent binds 445, the child never
needs the privilege.

Windows has no `ExtraFiles`, so there the child binds the address itself; the
isolation is the same, the privileged-port half is not.

## The rule that makes it honest

A child opens the image itself, so an image served *writable* by two protocols
would be two drivers over one file with **no lock between them** — exactly what
the [shared lock](locking.md) prevents inside one process.

That is refused, and the refusal says how to fix it:

```
scratch is writable over smb and webdav, and one process per protocol means
that many drivers writing one file with no lock between them. Say
`protocols = ["smb"]` on the share, or make it read_only, or do not isolate.
```

`protocols = [...]` on a share is useful on its own, not only under
`--isolate`: a share declared SMB-only is a share the WebDAV process is never
told about. A `serve` block that would end up carrying nothing is refused too,
before any image is opened, naming the shares that were kept from it and why.
