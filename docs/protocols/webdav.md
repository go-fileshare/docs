# WebDAV — Basic, or a bearer token

```hcl
serve "webdav" { addr = "0.0.0.0:8080" }
```

HTTP Basic, over whatever TLS the transport gives it.

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

!!! danger "Only WebDAV"
    SMB authenticates with NTLMv2, SFTP with a key or a certificate, NFS with
    nothing at all: none of them has anywhere to put an `Authorization` header.
    That is a fact about the protocols, not a limit of this program, and a
    configuration naming a provider without serving WebDAV is **refused rather
    than started**.

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

## The root, and uploads in pieces

**The root lists the shares** (since v0.29.0). A browser at `/` gets a page of
the shares it may open; a `PROPFIND` of `/` gets them as collections (`207`),
so a file manager can mount the whole server rather than one share. Depth
`infinity` is refused (`403`, `propfind-finite-depth`, as RFC 4918 allows), and
so is anything that would write at the root (`405`).

**A large upload can go in pieces** (since v0.29.0). One `PUT` is held in
memory whole, up to 1 GiB, so a client that cannot stream — a browser, the
[explorer](../ui.md) — sends an empty `PUT`, then each piece with a
`Content-Range` that starts exactly where the last one ended, then a `MOVE` to
the real name. A range that starts past the end is still `416`: it would leave
bytes nobody sent.
