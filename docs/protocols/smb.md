# SMB — NTLMv2

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

The password **never crosses the wire**: a client sends a proof computed from
it. The share tells a reader they are one — in the access mask, before they
try, rather than by failing a write later.

## What a directory must hold

NTLMv2 needs the password itself or its MD4 (`sambaNTPassword`, an `nt_hash`
column). **Nothing else will do.**

A directory that only *checks* passwords — an LDAP bind, a bcrypt column —
cannot answer SMB, however good the check is, because the server has to compute
the same proof the client did. That is a property of the protocol, not a
limitation of this program.

[`check`](../configuration/check.md) prints a `-` in the SMB column for such a
person, at configuration time, rather than leaving them to discover it at a
mount.

## A privileged port without a privileged server

Under [`--isolate`](../operations/isolation.md) on Unix, the parent binds 445
and passes the listener to the child, so the process actually speaking SMB
never needs the privilege. Windows has no `ExtraFiles`, so there the child
binds the address itself; the isolation is the same, the privileged-port half
is not.

## Verified against a client this project did not write

[go-smb2](https://github.com/hirochachacha/go-smb2) drives twenty concurrent
reads through SMB while twenty run through WebDAV, under `-race`, in CI. macOS
mounts a share over SMB while `curl` writes to the same image over WebDAV, and
reads back what WebDAV wrote.
