# WebDAV — Basic, or a bearer token

```hcl
tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true
}
```

HTTP Basic, or a bearer token — over [TLS](../security/tls.md), from files or
from ACME.

!!! danger "Not in the clear, since v0.9.0"
    HTTP Basic is the password on every request, and a bearer token is as good
    as one. WebDAV **without TLS**, on an address other machines can reach,
    while anybody authenticates, is **refused at startup** —
    `serve "webdav" { addr = "0.0.0.0:8080" }`, the example this page used to
    show, no longer starts. When a proxy terminates TLS in front of it, say so:

    ```hcl
    serve "webdav" {
      addr      = "10.0.0.5:8080"
      plaintext = true
    }
    ```

    Loopback, and a server with nobody to authenticate, are unaffected. See
    [TLS](../security/tls.md#webdav-is-not-served-in-the-clear).

!!! note "404, not 403"
    A share a person may not use answers **404**. It is not confirmed to exist.

## A token, over the one protocol that can carry one

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
}
```

A browser has a token and no password. So WebDAV accepts `Authorization:
Bearer`, and the challenge it sends offers **both** — a client picks the one it
can answer. The token is verified by
[go-authn/oidc](https://github.com/go-authn/oidc): signature, issuer, audience,
expiry.

!!! danger "A bearer token: only WebDAV"
    SMB authenticates with NTLMv2, SFTP with a key or a certificate, NFS with
    nothing at all, and S3 with a SigV4 signature: none of them has anywhere to
    put an `Authorization` header. That is a fact about the protocols, not a
    limit of this program.

    The provider's word reaches **SFTP** in a certificate instead — see
    [SFTP](sftp.md#people-the-identity-provider-vouches-for). A configuration
    naming a provider and serving neither WebDAV nor such an SFTP is **refused
    rather than started**.

!!! danger "A token says who the provider thinks somebody is. It does not say this server has a share for them."
    A valid token for a name no source here knows is refused — the safe reading
    of *I do not know you* is not *you are allowed*.

A site where the provider **is** the directory says so:

```hcl
oidc {
  issuer    = "https://login.example.org"
  audience  = "fileshare"
  trust_all = true      # everybody that provider vouches for, not just these people
}
```

There is **no login flow** here: no redirect, no client secret, no cookies.
This is the resource server.

**The name** a token gives is `preferred_username`, or the claim
`username_claim` names. Without `preferred_username`, it is the email only if
the provider says `email_verified: true`, and otherwise `sub`. With
`username_claim = "email"`, a token whose email is not verified is refused —
over WebDAV, and for an opkssh login over SFTP alike. Until it is verified,
the email is only what the person typed, and a signed token that carried it
would bind as an address nobody checked (OpenID Connect Core 5.1;
go-authn/oidc v0.2.0, fileshare v0.16.3).

Which of the provider's people get a share — groups, named people, whole
institutions — is written with `oidc:` rules and `domains`; see
[people the identity provider names](../configuration/identity.md#people-the-identity-provider-names-not-this-file).

## A token outlives the person, unless something says otherwise

A token is verified here, on its own, and is valid until it expires — whatever
happened to the person since. No revocation list can name it. An
[`ssf` block](../security/shared-signals.md) makes a CAEP `session-revoked`
from the provider void every token issued (`iat`) before it.
