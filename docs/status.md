# Status

## Verified

These are things a client that this project did not write was made to do, not
assertions about the code.

- **macOS** mounts a share over SMB while `curl` is writing to the same image
  over WebDAV, and reads back what WebDAV wrote. Bob's WebDAV write to a share
  he may only read is `403`; a wrong password is `401`.
- **go-smb2** — a client this project did not write — drives twenty concurrent
  reads through SMB while twenty run through WebDAV, under `-race`, in CI.
- The access rules are checked **through every protocol** rather than in the
  configuration alone: alice writes and bob does not, over SMB and over WebDAV,
  and NFS is not offered the share at all.
- SFTP certificates are verified against **OpenSSH's own client**, which also
  refuses the same key once its certificate is moved aside.

## Not yet

**S3** — the fifth protocol, and the one that would make an image reachable
from anything that speaks object storage. It needs a
[`go-filesystems/s3`](https://github.com/go-filesystems/s3) to exist first:
SigV4 is stdlib arithmetic, and the union tree in `unionfs.go` is already the
shape a bucket list wants.

## Measured, and stated as measured

Two claims in these pages are insurance rather than reproductions of a defect,
and are written that way in the source too:

- Eight goroutines writing through an unwrapped fat32 driver under `-race`
  produce no race today. The [shared lock](operations/locking.md) is insurance
  on a contract — `go-filesystems/interface` promises nothing about concurrent
  path-based calls — not a fix for an observed failure.
- The [build-tag sizes](operations/build-tags.md) were measured, including the
  13.2 MB that `hashicorp/go-plugin` costs on its own, which is what decided
  against a plugin framework.
