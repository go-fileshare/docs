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

**Writes over S3.** `PUT` and `DELETE` answer 403. The library can write, but
a share a person may only read has to refuse at the same place SFTP does, and
that is not wired. Refusing beats a half-written object.

**An OIDC token over S3.** Tokens work over [WebDAV](protocols/webdav.md) and
nowhere else. The usual route for S3 is STS `AssumeRoleWithWebIdentity`,
which exchanges the token for temporary credentials the client then signs
with — a different mechanism from accepting a bearer token, and not
implemented.

[S3 itself](protocols/s3.md) has shipped: a share is a bucket, served over
the same per-user tree SFTP uses.

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
