---
title: "What revokes what"
linkTitle: "Security"
weight: 40
description: "Which mechanism takes back each kind of credential fileshare accepts, and why every one of them fails closed."
tags: [security, revocation]
---

A credential is checked when it is presented. Taking a person back afterwards —
a staff member who left, an account disabled in go-authn/bridge, a whole IdP
switched off — has to reach **every** credential they were given, and the
sessions those already opened. Each kind of credential here is reached by a
different mechanism, because each is issued and checked differently:

| credential | protocol | what takes it back | added in |
|---|---|---|---|
| a password, NT hash or application password in a `users` directory (SQL, LDAP) | SMB, WebDAV, S3 | the row removed, then a [directory reload]({{< relref "/administration/reload.md" >}}): a new generation, and the connections it opened closed | v0.10.0 |
| a share grant | every protocol | the [admin API]({{< relref "/administration/_index.md" >}})'s `Revoke`, or `DisableShare`: a new generation, connections closed | v0.8.0 |
| an SSH certificate the provider's CA signed | SFTP | its [KRL]({{< relref "/security/revocation-lists.md" >}}), checked at login **and at every operation**; and [shared signals]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |
| an OpenPubkey (opkssh) certificate | SFTP | no list can name it; [shared signals]({{< relref "/security/shared-signals.md" >}}), and `opkssh_max_age` | v0.13.0 |
| an access token | WebDAV | no list can name it; [shared signals]({{< relref "/security/shared-signals.md" >}}) | v0.13.0 |
| an X.509 client certificate naming a person | NFS | its [CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}), at the next call; and [shared signals]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |

Without the mechanism in the third column, a credential stays good until it
expires — and for a certificate or a token, that window is its lifetime.

## Every one of them fails closed

A revocation list that cannot be fetched, a copy of one older than its
`max_age`, a CRL past its own `NextUpdate`, a shared-signals transmitter not
heard from within `max_age`: none of these is read as "nothing is revoked". It
is "this server does not know", and the credentials that mechanism governs are
**refused** until it knows again.

{{< callout type="error" >}}
**The price is availability, and it is paid on purpose**

A revocation check that lets everything through when the list is
unreachable is the one an attacker who can block the list defeats — the
reason browsers' soft-fail OCSP was called "a seat-belt that snaps when you
crash". So the list, or the transmitter, must be served as reliably as the
logins it governs, and the metrics that say it is going stale are the ones
to [alert on]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

A copy is replaced only by a list that parsed: a list that arrives broken leaves
the last good one in place, still counting towards `max_age` from when **it**
was fetched. And a list is fetched over **HTTPS only** — a list an attacker
on the path can replace revokes nothing — so a plain-HTTP URL is refused at
startup.

## Transport

The credentials themselves must not cross a network in the clear. That is
[TLS]({{< relref "/security/tls.md" >}}): WebDAV, S3 and NFS over TLS, a certificate from files or ACME,
and — since v0.9.0 — **WebDAV with passwords refused in the clear**.
