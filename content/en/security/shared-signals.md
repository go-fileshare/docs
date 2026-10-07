---
title: "Revoking what no list covers: shared signals"
linkTitle: "Shared signals"
weight: 40
description: "Revoking access tokens, OpenPubkey certificates and other federated credentials with CAEP session-revoked events from a shared signals transmitter."
tags: [security, revocation, shared signals]
---

A certificate has a revocation list — the [KRL]({{< relref "/security/revocation-lists.md" >}}), the
[CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}). An **access token** fileshare
verifies on its own, and an **OpenPubkey certificate** the person's own key
signed, do not: they are valid until they expire, whatever happened to the
person since.

Since v0.13.0, the [OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html)
closes that, with a [CAEP](https://openid.net/specs/openid-caep-1_0-final.html)
**`session-revoked`** event:

```hcl
ssf {
  transmitter        = "https://bridge.example.org"          # its issuer
  audience           = "https://files.example.org"           # what this server is to it
  client_id          = "fileshare"                           # OAuth client credentials,
  client_secret_file = "/etc/fileshare/ssf.secret"           # scope "ssf" (or token_file)
  state_file         = "/var/lib/fileshare/revocations.json"
  max_age            = "10m"                                 # default; retain = "192h"
}
```

go-authn/bridge sends one when a person or an IdP is disabled; fileshare
**polls** for it (RFC 8936, the SSF default) and keeps, per person, **when** it
happened.

## Everything issued before is refused, on every protocol

From that moment, every federated credential **issued before** it is refused —
and **the sessions those opened stop being served**, through the same gates as
the KRL's. What "issued" means depends on what carried it:

| protocol | credential | its issue time |
|---|---|---|
| WebDAV | an access token | its `iat` |
| SFTP | a certificate the provider's SSH CA signed | its validity start (`ValidAfter`) |
| SFTP | an OpenPubkey (opkssh) certificate | the ID token's `iat` |
| NFS | a client certificate ([identity = "certificate"]({{< relref "/security/nfs-certificates.md" >}})) | its `NotBefore` |

What the provider issues **after** is theirs: a person re-enabled is not locked
out.

{{< callout type="info" >}}
**The same second counts as before**

CAEP's `event_timestamp` and a token's `iat` are whole seconds, and a
credential issued **at or before** the revocation is refused — so one issued
in the same second as a revocation is refused too, and a person re-enabled
right after being disabled gets working credentials from the next second on.
That is the conservative side, on purpose. An issue time that is not known
is not "long ago": it is refused once there is any revocation for that
person.
{{< /callout >}}

## Who an event is about

The subject is RFC 9493's **`account`** (`acct:user@domain`, the name the shares
use), **`iss_sub`**, **`email`**, or **`aliases`** of them.

**An IdP disabled whole** arrives as a CAEP *tenant* subject with the IdP's
scopes in the event: everyone whose name is `@` one of them — compared whole,
`evil-univ-a.fr` is not `univ-a.fr` — and whose credential was issued before,
is refused, **including people the provider no longer remembers**.

## How the receiver authenticates

With **OAuth client credentials** (RFC 6749 §4.4, scope `ssf`): `client_id` and
`client_secret_file`, the token fetched from the transmitter's own token
endpoint — found in its OpenID configuration, or `token_url` — and renewed
before it expires. `token_file` is there instead for transmitters that hand out
a long-lived bearer token. One or the other: both, or neither, is refused.

| field | default | |
|---|---|---|
| `transmitter` | | its issuer, **HTTPS only** |
| `audience` | | what this server is to the transmitter; required, or a SET addressed to another receiver would be believed here |
| `client_id`, `client_secret_file` | | OAuth client credentials, together |
| `token_url` | discovered | HTTPS only, or the secret crosses the network in the clear |
| `token_file` | | a long-lived token instead of client credentials |
| `state_file` | | **required**: where revocations are written down |
| `ca_file` | the system's | pins the transmitter's authorities |
| `max_age` | `10m` | how long without hearing from the transmitter before federated credentials are refused; at least a minute |
| `retain` | `192h` | how long a revocation is kept; at least 169h (since v0.14.0) |

A revocation is **written down before it is acknowledged**, so an acknowledged
one survives a restart; and it is kept for `retain` — longer than any credential
it could void (192h is past the longest-lived credential the provider issues, a
168h SSH certificate).

An `ssf` block with nothing federated to revoke — no `oidc` block, no NFS
`identity = "certificate"` — is refused.

The transport is [github.com/hstern/go-ssf](https://github.com/hstern/go-ssf):
discovery, the RFC 8936 poller, the SET's JWS layer. What an event **means** is
fileshare's.

## It fails closed

{{< callout type="error" >}}
**While the transmitter is silent, federated credentials are refused**

Like the KRL: while the transmitter has not answered a poll within
`max_age`, **every federated credential** is refused, because "no
revocation arrived" and "none could" look the same.
`fileshare_ssf_last_heard_seconds` is the metric to alert on;
`fileshare_ssf_revoked_subjects` is how many revocations are kept. See
[health and metrics]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

## Checked end to end

go-authn/bridge's interop lane
([#12](https://github.com/go-authn/bridge/pull/12),
[#17](https://github.com/go-authn/bridge/pull/17)) judges fileshare v0.13.0
against the real bridge: the KRL, the CRL over NFS, WebDAV tokens revoked over
SSF, an IdP disabled by scope, and opkssh over SSF — the case no revocation list
reaches.
