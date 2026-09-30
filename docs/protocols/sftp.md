# SFTP — keys, or certificates

Every client already has it: `sftp` ships with OpenSSH, the Finder and GNOME
mount it, editors speak it. It is also the one protocol here whose shape does
not fit — a person logs in and lands in **one** filesystem, not a list of
shares.

So the shares become the top-level directories of a tree built for whoever just
authenticated:

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

A share alice may not use **is not a directory alice can see**, and a share she
may only read refuses her writes *in the tree*, before a driver that would have
allowed them.

!!! note "A rename across two shares is refused"
    It would be a copy and a delete over two images, and that is not what
    rename promises anywhere.

## Configuration

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # optional

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # nothing here: her certificate is her credential
```

With `trusted_user_ca_file`, a certificate signed by that authority and naming
the user among its principals is enough — so a person's access is **issued and
expires elsewhere**, and no file here is edited when somebody joins or leaves.

The signature, the validity window and the principals are checked by
`x/crypto/ssh`'s `CertChecker`; verified against **OpenSSH's own client**,
which also refuses the same key once its certificate is moved aside.

!!! warning "Without `host_key_file` a fresh identity is generated at every start"
    The server says so, and every client that has seen it before will warn
    about a changed key — which is the client doing its job.

## People the identity provider vouches for

Since v0.11.0 an `oidc` block reaches SFTP too — not with a token, which SSH has
nowhere to put, but with a **certificate**:

```hcl
oidc {
  issuer           = "https://login.example.org"
  audience         = "fileshare"
  ssh_ca_file      = "/etc/fileshare/bridge-ca.pub"   # go-authn/bridge's ssh_ca
  opkssh_client_id = "opkssh"                         # OpenPubkey logins
  opkssh_max_age   = "24h"                            # 12h, 24h, 48h, 1week
}
```

- **A certificate the provider's SSH CA signed** — `bridge ssh-cert` writes one
  after a login through the federation. Its principal is the person, its
  `groups@go-authn.org` extension their groups, so
  [`oidc:groups:` rules](../configuration/identity.md#people-the-identity-provider-names-not-this-file)
  apply. It is **not** `trusted_user_ca_file`, whose certificates are about
  local accounts: a local authority's certificate claiming the provider's groups
  is read as the local account it names.
- **An OpenPubkey certificate**, as `opkssh login` writes: signed by the user's
  own key, with an ID token that commits to that key. It is checked as
  `opkssh verify` checks it — the
  [openpubkey](https://github.com/openpubkey/openpubkey) verifier (the
  provider's signature against its published keys, the nonce commitment, the
  client ID, the age), then the certificate's key against the token's — and the
  SSH user name must be the token's username, because there is no `auth_id` file
  here to map one to the other: `sftp alice@univ-example.fr@files.example.org`.
  `opkssh_client_id` should be a client ID of its own, never another
  application's: the ID token travels to every server the person logs into.

To see the groups a certificate carries:

```sh
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub     # the Extensions section
```

and the server says, at every federated login,
`sftp: alice@univ-a.fr, vouched for by the provider, in groups [...]`.

!!! danger "Vouched for is not admitted"
    Somebody the provider vouches for is still a stranger here unless a rule
    names them or `trust_all` says the provider is the directory — the same test
    a token passes over WebDAV. A provider certificate with **no principal**,
    valid for *anybody* by the format's own definition, is refused.

`-tags noopenpubkey` leaves the OpenPubkey verifier out (about 2 MB); a
configuration with `opkssh_client_id` in such a build is refused.

## Taking a certificate back

!!! danger "A certificate is checked at login"
    Revoking the person at the provider does not, by itself, revoke a
    certificate already issued: it opens SFTP until it expires, and an SFTP
    session already open stays open — these people are not in the directory, so
    no [reload](../administration/reload.md) concerns them. The window is the
    certificate's lifetime: bridge's `ssh_ca { validity }` (12h by default, at
    most 168h, never past the IdP session's end) and `opkssh_max_age` here. Keep
    it as short as the clients' re-login allows.

Two mechanisms close that window:

- the provider's **[KRL](../security/revocation-lists.md)** (`ssh_krl_url`):
  a revoked certificate is refused at login, and every operation of a session it
  opened asks the list again, open files included;
- **[shared signals](../security/shared-signals.md)** (`ssf`): everything
  the provider issued the person before a CAEP `session-revoked` is refused —
  the only way to reach an OpenPubkey certificate, which is in no list.

Both fail closed.

## No password

A client that prompts for one is doing the thing keys exist to avoid.
