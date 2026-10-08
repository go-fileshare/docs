---
title: "SFTP — keys, or certificates"
linkTitle: "SFTP: keys, or certificates"
weight: 30
description: "SFTP with public keys or SSH certificates: local and provider authorities, OpenPubkey, domain grants, source-address, and revocation."
tags: [protocols, sftp, ssh]
---

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

{{< callout type="info" >}}
**A rename across two shares is refused**

It would be a copy and a delete over two images, and that is not what
rename promises anywhere.
{{< /callout >}}

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

{{< callout type="warning" >}}
**Without `host_key_file` a fresh identity is generated at every start**

The server says so, and every client that has seen it before will warn
about a changed key — which is the client doing its job.
{{< /callout >}}

## Certificates meant for this host: the domain grant

Since v0.22.0. The [EuroHPC Federation Platform's SSH CA](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-overview/)
(GÉANT / MyAccessID) signs certificates that every site of the federation
trusts. Each certificate says which hosts it is meant for, in the
`ssh-domain-grant@core.aai.geant.org` extension. sshd ignores that extension,
and so did fileshare until v0.22.0: a certificate granted for somebody else's
machines was as good here. With `ssh_domains`, the grant is read, parsed and
matched by [go-authn/sshcert](https://github.com/go-authn/sshcert) as the
specification says. A certificate whose grant names none of this host's names
is refused.

```hcl
trusted_user_ca_file = "/etc/fileshare/efp-ssh-ca.pub"
ssh_domains          = ["files.example.org", "sftp.example.org"]
# ssh_accept_ungranted = true   # only if a trusted CA never writes a grant
```

| key | |
|---|---|
| `ssh_domains` | this host's names, as a grant names them. A certificate is accepted if one pattern of its grant matches one of them. `*` is **one or more characters within one label**: `*.example.org` grants `files.example.org` but not `a.files.example.org`, and `files-*.example.org` does not grant `files-.example.org`. The comparison ignores case. These are host names, not patterns, each listed once, and at least one authority (`trusted_user_ca_file`, or the `oidc` block's `ssh_ca_file`) must be trusted. |
| `ssh_accept_ungranted` | let in a certificate with **no** grant. Off by default; without `ssh_domains` it is refused as meaningless. |

| certificate | `ssh_domains` | `+ ssh_accept_ungranted` | no `ssh_domains` |
|---|---|---|---|
| granted one of `ssh_domains` | in | in | in |
| granted other hosts only, or `[]` | refused | refused | in |
| no grant | refused | in, logged | in |
| a grant that does not parse (`null`, `[1]`, not compact...) | refused | **refused** | in |

Without `ssh_domains` nothing is read and nothing changes. The grant is read on
the certificates an **authority** signed: `trusted_user_ca_file`'s, and the
provider's (`ssh_ca_file`). An OpenPubkey certificate is signed by the user's own
key, so a grant in it would be the user's word about themselves. Its audience is
`opkssh_client_id` instead. Plain keys carry no grant and are not concerned.

Each refusal is logged with the certificate's serial, its key ID and the reason:

```
sftp: alice: certificate 42 ("alice@efp") refused: its domain grant [login.example.eu] names none of this host's names [files.example.org sftp.example.org]
sftp: alice: certificate 43 ("staff") refused: it has no domain grant (ssh-domain-grant@core.aai.geant.org), and ssh_domains requires one
```

Each decision is also counted in `fileshare_sftp_domain_grant_total{result}`,
with the result `granted`, `accepted_ungranted`, `refused_not_granted`,
`refused_absent` or `refused_malformed`. A client offers a certificate before it
proves it holds the key, so these count attempts, not people.

{{< callout type="error" >}}
**Why fail-closed**

Take a host that trusts a second authority: its own CA for staff, a test
CA, EFP's staging CA appended to the same file, or a go-authn/bridge client
without grants. That authority's certificates carry no grant. If a missing
grant meant "no restriction", the filter would filter nothing on that host.

Here, a certificate with no grant is refused unless `ssh_accept_ungranted`
says otherwise. A grant that is present but does not parse is refused
whatever it says: an authority that wrote one meant to restrict the
certificate. go-authn/sshcert walks through the
[multi-CA scenario](https://github.com/go-authn/sshcert#why-fail-closed-the-multi-ca-scenario).
{{< /callout >}}

### With EFP's CA

`trusted_user_ca_file` is OpenSSH's `TrustedUserCAKeys`. Put in it the key EFP
publishes at <https://sshca.my-eurohpc.eu/config>, as EFP's
[trust page](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-trust/) says.
That URL returns `{"PublicKey": "ssh-ed25519 ..."}`; the file holds the value
of `PublicKey` on a line of its own:

```sh
curl -s https://sshca.my-eurohpc.eu/config | jq -r '.PublicKey' > /etc/fileshare/efp-ssh-ca.pub
```

Then set `ssh_domains` to the names your hosting entity's grant uses. An EFP
certificate's one principal is the MyAccessID identifier
(`<id>@myaccessid.org`), so that is the user name it logs in as. A `user` block
of that name, with no credential of its own, receives it:

```hcl
user "u1234@myaccessid.org" {}
```

### With go-authn/bridge

A [go-authn/bridge](https://github.com/go-authn/bridge) client (v0.19.0 or
later) with `ssh_certificates = true` and
`ssh_domain_grants = ["files.example.org"]` issues certificates granted for
this host. With `ssh_principal_claim = "voperson_id"` its one principal is the
person's `voperson_id`, as in the EFP profile, and that is the `user` block to
write here. Trust bridge's CA the way this server trusts any:

- as `trusted_user_ca_file`, for local accounts. bridge serves its key in EFP's
  format at `/ssh/config`, so the `curl | jq` line above works with bridge's URL;
- or as the `oidc` block's `ssh_ca_file`, for
  [people the provider vouches for](#people-the-identity-provider-vouches-for).

bridge's other clients write no grant. If their certificates must keep working
here, that is what `ssh_accept_ungranted` is for. But it lets in **every**
ungranted certificate of every trusted authority, so prefer giving those
clients a grant.

### A certificate pinned to an address

bridge's `ssh_source_address` writes the `source-address` critical option, as
`ssh-keygen -O source-address=...` does. A certificate that carries one is
accepted **from those addresses only**, with `ssh_domains` as without it. A
login from an address the certificate allows is let in, provided its grant also
names this host when `ssh_domains` is set. A login from any other address is
refused.

| fileshare | `trusted_user_ca_file` | `oidc` `ssh_ca_file`, and OpenPubkey |
|---|---|---|
| before v0.22.0 | enforced | **refused from every address** |
| v0.22.0, without `ssh_domains` | enforced | **refused from every address** |
| v0.22.0, with `ssh_domains` | **refused from every address** | **refused from every address** |
| v0.22.1, with or without `ssh_domains` | enforced | enforced |

"Refused from every address" fails closed: the certificate's restriction was
never loosened, only the login lost. Before v0.22.1, go-filesystems/sftp's
sshd refused any critical option on certificates it handed to fileshare to
decide; that covers the provider's, OpenPubkey ones, and, under `ssh_domains`,
every certificate. v0.22.1 is built on go-filesystems/sftp v0.5.1, whose sshd
accepts `source-address` there and enforces it on every certificate path,
against the address the connection comes from, IPv4 or IPv6.

Any other critical option (`force-command`, `verify-required`, ...) is refused,
since this server does not act on it.

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
  [`oidc:groups:` rules]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})
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

{{< callout type="error" >}}
**Vouched for is not admitted**

Somebody the provider vouches for is still a stranger here unless a rule
names them, `trust_all` says the provider is the directory, or
`local_names = true` says the provider's names are this server's and a local
account has theirs — the same test a token passes over WebDAV. A provider certificate with **no principal**,
valid for *anybody* by the format's own definition, is refused.
{{< /callout >}}

`-tags noopenpubkey` leaves the OpenPubkey verifier out (about 2 MB); a
configuration with `opkssh_client_id` in such a build is refused.

## Taking a certificate back

{{< callout type="error" >}}
**A certificate is checked at login**

Revoking the person at the provider does not, by itself, revoke a
certificate already issued: it opens SFTP until it expires, and an SFTP
session already open stays open until then — these people are not in the directory, so
no [reload]({{< relref "/administration/reload.md" >}}) concerns them. The window is the
certificate's lifetime: bridge's `ssh_ca { validity }` (12h by default, at
most 168h, never past the IdP session's end) and `opkssh_max_age` here. Keep
it as short as the clients' re-login allows. Since v0.17.0 an opkssh
session that is already open also ends at `opkssh_max_age`, whatever the
certificate (which the person signs themselves) says.
{{< /callout >}}

Two mechanisms close that window:

- the provider's **[KRL]({{< relref "/security/revocation-lists.md" >}})** (`ssh_krl_url`):
  a revoked certificate is refused at login, and every operation of a session it
  opened asks the list again, open files included;
- **[shared signals]({{< relref "/security/shared-signals.md" >}})** (`ssf`): everything
  the provider issued the person before a CAEP `session-revoked` is refused —
  the only way to reach an OpenPubkey certificate, which is in no list.

Both fail closed.

## No password

A client that prompts for one is doing the thing keys exist to avoid.

## Transfer size and open files

fileshare answers OpenSSH's `limits@openssh.com` (since v0.25.0), so
OpenSSH's `sftp` reads and writes up to 255 KiB per request instead of
32 KiB: about +25% measured with OpenSSH 10.3. One session may hold at most
1024 open files and directories at once; on a directory share each is a
descriptor of the host, and the bound keeps one user from spending all of
them.
