# Status

These pages describe **fileshare v0.13.0**.

## What each release added

| | |
|---|---|
| v0.7.0 | a restricted share over NFS, with a [`kerberos` block](protocols/nfs.md#the-refusal-that-used-to-be-permanent) |
| v0.8.0 | the [admin API](administration/index.md) over gRPC; [directory shares](configuration/shares.md#a-directory-not-only-an-image); [`/healthz`, `/readyz`, `/metrics`](administration/health.md) |
| v0.9.0 | [TLS](security/tls.md) for WebDAV, S3 and NFS, from files or ACME; **WebDAV with passwords refused in the clear** |
| v0.10.0 | [reading the directory again](administration/reload.md): `reload = "5m"`, `SIGHUP`, `ReloadDirectory` |
| v0.11.0 | [SFTP for people the identity provider vouches for](protocols/sftp.md#people-the-identity-provider-vouches-for): its SSH CA, and opkssh |
| v0.12.0 | revocation: a [KRL](security/revocation-lists.md) for the provider's SSH certificates, a CRL for [NFS identities from certificates](security/nfs-certificates.md) |
| v0.13.0 | [shared signals](security/shared-signals.md): a CAEP `session-revoked` voids everything the provider issued before, on every protocol |

!!! warning "Upgrading past v0.9.0"
    A configuration serving WebDAV without TLS on an address other machines can
    reach, with anybody to authenticate, **no longer starts**. Add `tls = true`
    and a `tls` block, or `plaintext = true` behind a TLS-terminating proxy. See
    [TLS](security/tls.md#webdav-is-not-served-in-the-clear).

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
- NFS identities from certificates were measured with a **real Linux client**
  (kernel 6.17, ktls-utils 0.9) in go-filesystems/nfs's CI — which is how
  [`MNT` in the clear and `tlshd`'s pitfalls](security/nfs-certificates.md#measured-with-a-real-linux-client)
  were found.
- The KRL's unsigned form is a measurement: OpenSSH 9.6 and 10.3 read a signed
  KRL and skip the signature (go-authn/krl, against both).
- **go-authn/bridge's interop lane** ([#12](https://github.com/go-authn/bridge/pull/12),
  [#17](https://github.com/go-authn/bridge/pull/17)) judges fileshare v0.13.0
  against the real bridge: the KRL, the CRL over NFS, WebDAV tokens revoked over
  shared signals, an IdP disabled by scope, and opkssh over shared signals.

## Not yet

**Writes over S3.** `PUT` and `DELETE` answer 403. The library can write, but
a share a person may only read has to refuse at the same place SFTP does, and
that is not wired. Refusing beats a half-written object.

**An OIDC token over S3.** Tokens work over [WebDAV](protocols/webdav.md) and
nowhere else. The usual route for S3 is STS `AssumeRoleWithWebIdentity`,
which exchanges the token for temporary credentials the client then signs
with — a different mechanism from accepting a bearer token, and not
implemented.

**`--isolate` with an `admin` or a `metrics` block.** Refused: there is no one
process an API change could be applied to. See
[one process per protocol](operations/isolation.md#not-with-an-admin-or-a-metrics-block-yet).

**A person per user on a shared Linux NFS client, from a certificate.** A Linux
client attaches the certificate to a mount, so every user of the mount is the
person it names. That is a property of the client, and the answer there is
`sec=krb5`. See [what it cannot promise](security/nfs-certificates.md#what-it-cannot-promise).

[S3 itself](protocols/s3.md) has shipped: a share is a bucket, served over
the same per-user tree SFTP uses.

## Measured, and stated as measured

Claims in these pages that are insurance or measurement rather than
reproductions of a defect, and are written that way in the source too:

- Eight goroutines writing through an unwrapped fat32 driver under `-race`
  produce no race today. The [shared lock](operations/locking.md) is insurance
  on a contract — `go-filesystems/interface` promises nothing about concurrent
  path-based calls — not a fix for an observed failure.
- The [build-tag sizes](operations/build-tags.md) were measured, including the
  13.2 MB that `hashicorp/go-plugin` costs on its own, which is what decided
  against a plugin framework — and the 11.7 MB gRPC costs now that the admin API
  brings it, which is why `nogrpc` exists.
- A group-tag certificate extension of its own was designed and **measured**
  against Go's X.509 parser, which refused the whole certificate; the groups
  travel as [tag URIs](security/nfs-certificates.md#what-the-certificate-carries)
  instead.
