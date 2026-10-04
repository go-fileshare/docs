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
  ssh_krl_url        = "https://bridge.example.org/ssh/krl"   # or ssh_krl_file
  ssh_krl_state_file = "/var/lib/fileshare/ssh.krl.state"    # the order, across a restart
  ssh_krl_max_age    = "1h"                                  # default; ssh_krl_refresh = "1m"
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
| `ssh_krl_state_file` | | where the last list that verified is kept, so a restart remembers the order |

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

## Signed and dated, from `ssh_krl_url`

Since v0.15.0, **a KRL from `ssh_krl_url` must be signed and say when it
expires**, as [go-authn/revocation](https://github.com/go-authn/revocation)
describes and go-authn/bridge serves from v0.10.0:

- **The signature:** `<url>.sig` is a detached SSHSIG signature by the
  provider's SSH CA (`ssh_ca_file`, which `ssh_krl_url` therefore requires), in
  the namespace `krl@go-authn.github.io`. It is fetched with `If-Match` on the
  list's ETag, and fetched again at once if the list was re-issued in between.
- **The expiry:** the `expires@go-authn.github.io` extension. Past it, the list
  is not current, however recently a `304` confirmed it.

HTTPS (`ssh_krl_ca_file` pins its authorities) authenticates the server that
answered, not the list: a mirror, a cache or a compromised web server could
serve an empty one, and nothing in a KRL alone says it is stale. A list that
does not verify is refused, and the last good copy is kept. A list placed by
hand with `ssh_krl_file` is trusted as the filesystem that holds it, and needs
neither signature nor expiry; one it carries is honoured.

!!! warning "Upgrading to v0.15.0"
    An `ssh_krl_url` serving an unsigned KRL, or one with no expiry, is no
    longer taken: the provider's certificates are then refused, as for any
    list that cannot be had. Run go-authn/bridge v0.10.0 or later first.

## Never backwards, across a restart too

Since v0.16.0, a list is refused if it is older than the one held: a lower
version (the KRL's version, a CRL's number), or the same version issued
earlier. With `ssh_krl_state_file` (`crl_state_file` for
[NFS](nfs-certificates.md#revocation-the-crl)), the last list that verified is
kept on disk, verified again at start, and orders the next one. On its own it
does not count as current until a list is fetched. Without it, a restart
forgets the order, and fileshare says so at start: an older list, still signed
and unexpired, would be taken.

These files, like `ssh_ca_file`, the CA files and the list files, **may not lie
inside a share**: whoever writes there would decide who gets in.

## A plain `sshd` next to fileshare

The same verification is available without fileshare: go-authn/revocation's
`revokd` fetches the lists, keeps only those that verify, keeps their order on
disk, writes `RevokedKeys` for `sshd`, and, when a list lapses, writes one that
revokes the CA itself, so `sshd` fails closed too.

## What it does not cover

**OpenPubkey (opkssh) certificates are not in any KRL** — nothing issued them
but the person's own key. They are bounded by `opkssh_max_age`, and taken back
by [shared signals](shared-signals.md). So is an access token over WebDAV.

The same mechanism — fetched, kept, failing closed — carries the **X.509 CRL**
of [NFS identities from certificates](nfs-certificates.md#revocation-the-crl).
