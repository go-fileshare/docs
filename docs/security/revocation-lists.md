# Revoking SSH certificates: the KRL

A certificate the identity provider's SSH CA signed — see
[SFTP for people the provider vouches for](../protocols/sftp.md#people-the-identity-provider-vouches-for)
— is checked at login. Disabling somebody in go-authn/bridge revokes their
tokens and deletes their application passwords, which a
[directory reload](../administration/reload.md) applies here; but without a
list, a certificate already issued would still open SFTP until it expires, and
an SFTP session already open would stay open.

Since v0.12.0 a list closes that:

```hcl
oidc {
  issuer          = "https://login.example.org"
  audience        = "fileshare"
  ssh_ca_file     = "/etc/fileshare/bridge-ca.pub"
  ssh_krl_url     = "https://bridge.example.org/ssh/krl"   # or ssh_krl_file
  ssh_krl_max_age = "1h"                                   # default; ssh_krl_refresh = "1m"
}
```

go-authn/bridge publishes the SSH certificates it has revoked — a person
disabled, an IdP disabled — as an OpenSSH **KRL** (`PROTOCOL.krl`, the list
`sshd`'s `RevokedKeys` reads), and fileshare keeps a copy of it, read with
[go-authn/krl](https://github.com/go-authn/krl).

## Refused at login, and in the middle of a session

A revoked certificate is refused at login, and **a session it already opened
stops being served**. SSH checks a certificate once, at the handshake; so every
operation of a federated SFTP session asks the KRL again, **open files
included**. The first operation after this server's copy of the list changes is refused.

## Fields

| field | default | |
|---|---|---|
| `ssh_krl_url` | | where the list is fetched, **HTTPS only** |
| `ssh_krl_file` | | or a file, by absolute path — one or the other, never both |
| `ssh_krl_ca_file` | the system's | pins the authorities the list's HTTPS server is checked against |
| `ssh_krl_refresh` | `1m` | how often it is fetched, at least a second |
| `ssh_krl_max_age` | `1h` | how old the last good copy may be before the certificates it governs are refused |

A `max_age` shorter than `refresh` is refused — every copy would be stale before
the next fetch. A KRL with no `ssh_ca_file` is refused too: it revokes the
provider's certificates, and without that CA there are none.

## It fails closed

!!! danger "While the KRL cannot be fetched, the provider's certificates are refused"
    While the list cannot be fetched, or its last good copy is older than
    `ssh_krl_max_age`, **every** certificate the provider's CA signed is
    refused: a revocation check that lets everything through when the list is
    unreachable is the one an attacker who can block the list defeats. The list
    must then be served as reliably as the logins it governs.
    `fileshare_revocation_list_age_seconds{list="ssh_krl"}` is the metric to
    alert on before `max_age` is reached; see
    [health and metrics](../administration/health.md#what-to-alert-on).

## Not signed, and why that is not a gap here

The KRL is fetched over HTTPS only, with `ssh_krl_ca_file` to pin its
authorities. It is not signed, because nothing checks a KRL's signature:
OpenSSH 9.6 and 10.3 read a signed one and skip the signature (measured by
go-authn/krl against both). The transport is what protects it.

## What it does not cover

**OpenPubkey (opkssh) certificates are not in any KRL** — nothing issued them
but the person's own key. They are bounded by `opkssh_max_age`, and taken back
by [shared signals](shared-signals.md). So is an access token over WebDAV.

The same mechanism — fetched, kept, failing closed — carries the **X.509 CRL**
of [NFS identities from certificates](nfs-certificates.md#revocation-the-crl).
