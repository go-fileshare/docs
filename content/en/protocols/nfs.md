---
title: "NFS — nothing, unless Kerberos or a certificate"
linkTitle: "NFS: nothing, unless Kerberos or a certificate"
weight: 50
description: "TODO"
tags: [protocols, nfs, kerberos]
---

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

{{< callout type="error" >}}
**NFSv3 on its own authenticates nobody**

`AUTH_UNIX` is a **claim**: the client says "uid 501" and the wire cannot
disagree with it. There is no encryption either.
{{< /callout >}}

So **a share that names who may use it is not exported over NFS**. The refusal
is printed at startup and in [`check`]({{< relref "/configuration/check.md" >}}), with the
reason:

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it
```

A share with no `allow` and no `writers` — anyone who connects, read-write — is
exported over NFS, because there is no rule there for the protocol to fail to
honour.

## The refusal that used to be permanent

A `kerberos` block makes NFS able to tell people apart, and a restricted share
is then served over it like anywhere else:

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5` carries a principal a **ticket proves**, rather than a uid the
client asserts.

{{< callout type="info" >}}
**The realm is compared, not just the name before the `@`**

Two realms can each have an `alice`, and only one of them is yours.
{{< /callout >}}

## Over TLS: the machine, not the person

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

Since v0.9.0 NFS is served over RPC-with-TLS (RFC 9289) with `tls = true` and a
[`tls` block]({{< relref "/security/tls.md" >}}). It encrypts; with `client_ca_file` it makes a
client present a certificate from that authority — which **host** is mounting.
The uid inside is still the one `AUTH_SYS` claims, so a restricted share is
**still refused** over it. TLS is offered, not required: a client that never
asks for it is still served the open shares.

## A person, from their certificate

Since v0.12.0 a certificate can name a **person** — as go-authn/bridge issues
one after an identity provider login — and then a share that names people is
served over NFS without kerberos:

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # required
}
```

{{< callout type="error" >}}
**A Linux client's certificate belongs to a mount**

Every user of that mount acts as the person the certificate names. Right on
a workstation one person uses; wrong on a machine several people log into —
use `sec=krb5` there.
{{< /callout >}}

What the certificate carries, the CRL that is required, and what was measured
with a real Linux client (`MNT` in the clear, `tlshd`'s pitfalls) are in
[NFS, with identities from certificates]({{< relref "/security/nfs-certificates.md" >}}).
