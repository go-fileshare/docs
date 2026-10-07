---
title: "What a protocol can promise"
linkTitle: "Protocols"
weight: 20
description: "Why each protocol can or cannot tell who is asking, and what that means for which shares it serves."
tags: [protocols]
---

The protocols do not agree about the one thing access control needs: whether
the server can tell **who** is asking.

| | |
|---|---|
| **[SMB]({{< relref "/protocols/smb.md" >}})** | NTLMv2. The password never crosses the wire, and the share tells a reader they are one — in the access mask, before they try. |
| **[WebDAV]({{< relref "/protocols/webdav.md" >}})** | HTTP Basic, over [TLS]({{< relref "/security/tls.md" >}}): refused in the clear on a reachable address, unless `plaintext = true` says a proxy terminates TLS in front of it. A share a person may not use answers 404, not 403: it is not confirmed to exist. |
| **[SFTP]({{< relref "/protocols/sftp.md" >}})** | A **public key**, or an **SSH certificate** from an authority you trust: the server never holds the secret, and with a certificate a person's access is issued and expires elsewhere. No password: a client that prompts for one is doing the thing keys exist to avoid. |
| **[OIDC]({{< relref "/protocols/webdav.md" >}})** (over WebDAV) | A **bearer token** an identity provider signed. Verified by [go-authn/oidc](https://github.com/go-authn/oidc): signature, issuer, audience, expiry. No other protocol here has anywhere to put one — S3 signs with SigV4, which has no field for a bearer token. Over [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}) the provider's word arrives in a certificate instead. |
| **[S3]({{< relref "/protocols/s3.md" >}})** | **SigV4**, header or presigned. The secret proves itself by computing an HMAC and never crosses the wire — so, like NTLMv2, the directory must HOLD the password rather than merely check it. A share is a bucket. |
| **[NFSv3]({{< relref "/protocols/nfs.md" >}})** | **Nothing**, on its own. `AUTH_UNIX` is a claim — the client says "uid 501" and the wire cannot disagree. A `kerberos` block lifts this, and so do [identities from certificates]({{< relref "/security/nfs-certificates.md" >}}). [TLS]({{< relref "/security/tls.md" >}}) encrypts it, and proves the machine, not the person. |

## The consequence, stated once

**A share that names who may use it is not exported over NFS** — unless
Kerberos is configured, or NFS takes
[identities from certificates]({{< relref "/security/nfs-certificates.md" >}}).

Not a warning, not an option: a configuration saying *photos belongs to alice*
and a protocol handing photos to whoever connects cannot both be honoured, and
quietly widening access is the worse of the two failures. The refusal is
printed at startup and in [`check`]({{< relref "/configuration/check.md" >}}), with the reason.

## What each one needs from a directory

Which protocols can serve a given person is decided by what their source can
prove, not by the configuration alone. See
[what a source can prove]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}).

## What is encrypted

WebDAV, S3 and NFS take `tls = true`; SMB 3 encrypts with its own keys, and SFTP
is SSH. See [TLS, and certificates from ACME]({{< relref "/security/tls.md" >}}).
