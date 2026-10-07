---
title: "NFS, with identities from certificates"
weight: 30
description: "TODO"
tags: [security, nfs, tls, revocation]
---

NFSv3 on its own authenticates nobody, which is why a share that names who may
use it is [not exported over NFS]({{< relref "/protocols/nfs.md" >}}) — unless something can
tell people apart. A `kerberos` block is one answer. Since v0.12.0, a client
certificate that **names a person** is another:

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file; required
  crl_state_file = "/var/lib/fileshare/nfs.crl.state"     # the order, across a restart
}
```

## What the certificate carries

After an identity provider login, go-authn/bridge issues a **short-lived X.509
client certificate** naming the person the way FreeBSD's `rpc.tlsservd -u` reads
it:

- the person: the SubjectAltName **otherName `1.3.6.1.4.1.2238.1.1.1`**, a
  UTF8String `user@domain`
  ([draft-cel-nfsv4-rpc-tls-othername](https://www.ietf.org/archive/id/draft-cel-nfsv4-rpc-tls-othername-04.html)
  describes it; its OIDs are not assigned yet);
- their groups: one SubjectAltName URI each,
  `tag:go-authn.github.io,2026:group:<group>` — a **tag URI** (RFC 4151: no
  registration, made for exactly this). FreeBSD's `rpc.tlsservd` reads the first
  identity otherName and skips every other SAN entry.

{{< callout type="info" >}}
**Why a tag URI, and not an extension of its own**

One was designed, under a UUID-derived arc (`2.25.<128-bit number>`), and
**measured**: Go's `x509.ParseCertificate` refuses the whole certificate —
*malformed extension OID field* — because an ASN.1 object identifier arc is
an `int` there. Every Go TLS server would have turned the certificate away.
A URI SAN needs no OID, and every parser reads it. Not `urn:x-…` either:
RFC 8141 took the experimental URN namespaces away, and a strict parser
refuses one.
{{< /callout >}}

## What every call is asked

With `identity = "certificate"`, a share that names people is served over NFS
**without kerberos**. Every call on it must arrive over TLS, with a certificate
that:

1. is **not in the CRL**;
2. names **exactly one** person;
3. names somebody this server admits **the way it admits a token** — a rule,
   `trust_all`, `domains`, see
   [people the identity provider names]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}});
4. names somebody **the share allows**.

And, with an [`ssf` block]({{< relref "/security/shared-signals.md" >}}), the certificate must have been
issued (its `NotBefore`) **after** any revocation of that person.

## Revocation: the CRL

**A CRL is required** — `crl_url` or `crl_file`. A person's certificate that
nothing can revoke outlives their removal by its whole lifetime.

| field | default | |
|---|---|---|
| `crl_url` | | HTTPS only |
| `crl_file` | | or an absolute path — one or the other |
| `crl_ca_file` | the system's | pins the authorities of the CRL's HTTPS server |
| `crl_refresh` | `1m` | how often it is fetched |
| `crl_max_age` | `1h` | how old the last good copy may be |
| `crl_state_file` | | where the last CRL that verified is kept, so a restart remembers which is newer |

It is fetched and kept like the [KRL]({{< relref "/security/revocation-lists.md" >}}), and **fails closed**
the same way; a CRL past its own **`NextUpdate`** counts as unknown whatever
`crl_max_age` allows. A CRL must be signed by the client CA — a CRL anybody
could have written revokes whatever they like, and, worse, un-revokes it. With
the CRL reachable, **the next call after it changes is refused**; the
certificate's lifetime bounds a revocation only when the CRL is not reachable.

Since v0.16.2 the signature is checked on the raw bytes **before** the CRL is
parsed, through [go-authn/revocation](https://github.com/go-authn/revocation):
parsed first, a large CRL signed by anybody cost memory and seconds before
being refused. What this server cannot read as a complete list is refused
too: a delta CRL, an unknown critical extension or entry extension, no CRL
number, no `NextUpdate`. A CRL with a lower number than the one held, or the
same number issued earlier, is refused like an older
[KRL]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}}).

A CRL field without `identity = "certificate"` is refused, and so is
`identity = "certificate"` without `tls = true` and `client_ca_file`: the
identity is in the certificate the client presents.

## What it cannot promise

{{< callout type="error" >}}
**A Linux client's certificate belongs to a MOUNT, not a person**

The server sees a **connection's** certificate, and a Linux client attaches
one to a mount (`tlshd`, the keyring serial on
`mount -o xprtsec=mtls,...`). **Every user of that mount acts as the person
the certificate names.** On a workstation one person uses, that is exactly
them; on a machine several people log into, it is whoever mounted — use
`sec=krb5` there.

This is why RFC 9289 alone refuses to promise user authentication, and why
plain [NFS over TLS]({{< relref "/security/tls.md#nfs-over-tls-proves-the-machine-not-the-person" >}})
still proves only the machine.
{{< /callout >}}

## Measured with a real Linux client

Kernel 6.17, ktls-utils 0.9, in go-filesystems/nfs's CI:

- **MOUNT's `MNT` arrives in the clear**, whatever `xprtsec=` says: the
  kernel's mount client has no TLS. It is answered, and every NFS call made
  without TLS is then refused — so a person the share does not allow sees
  *access denied* at the first access rather than at `mount`. **Nothing of a
  share crosses in the clear.**
- `tlshd` checks this server's certificate against the **system** trust store
  only — it ignores `x509.truststore` — and wants the client's certificate and
  key **owned by root, the key mode 600**. Otherwise it fails with
  *gnutls: Error in the certificate (-43)*, naming neither.
