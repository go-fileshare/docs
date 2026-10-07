---
title: "TLS, and certificates from ACME"
weight: 10
description: "TODO"
tags: [security, tls, acme]
---

Since v0.9.0, **WebDAV and S3 are served over HTTPS, and NFS over
RPC-with-TLS** (RFC 9289), with `tls = true` on their `serve` block and one
`tls` block saying where the certificate comes from.

## From files

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

The files are **reloaded when they change**, so a renewal written by another
tool is picked up without a restart. The certificate is
[go-authn/servercert](https://github.com/go-authn/servercert)'s.

## From an ACME CA

Let's Encrypt by default; or a CA that knows you through **external account
binding**, such as GÉANT TCS through HARICA:

```hcl
tls {
  acme {
    directory_url     = "https://acme-v02.harica.gr/acme/<uuid>/directory"   # the Server URL cm.harica.gr shows
    domains           = ["files.example.org"]
    cache_dir         = "/var/lib/fileshare/acme"
    eab_key_id        = "…"
    eab_hmac_key_file = "/etc/fileshare/tls/eab.key"   # a secret: a file, never the config
  }
}
```

| field | |
|---|---|
| `directory_url` | the CA's ACME directory; empty is Let's Encrypt |
| `domains` | the names to ask for (required) |
| `cache_dir` | where the account and the certificates are kept (required) |
| `email` | the account's contact, optional |
| `eab_key_id`, `eab_hmac_key_file` | external account binding; the HMAC key is a secret, so it is read from a file — base64url, as the CA hands it out, or the block is refused |
| `http_challenge` | where to answer http-01, e.g. `"0.0.0.0:80"` |

### Whether ACME can work is decided by how the CA checks the name

| challenge | the CA connects to | so |
|---|---|---|
| tls-alpn-01 (RFC 8737) | port **443** | a TLS protocol must be served on 443 |
| http-01 (RFC 8555 §8.3) | port **80** | `http_challenge = "0.0.0.0:80"` answers it — or the address a port 80 is forwarded to |
| none | nothing | a CA account with the domain **pre-validated** asks for no challenge: HARICA's enterprise EAB accounts for GÉANT TCS — so a server **nobody outside can reach** still gets its certificate |

The last row is the one that matters for a file server inside a campus network:
with a GÉANT TCS account whose domain the institution has already validated at
HARICA, no port has to be opened to the Internet at all.

A certificate is asked for at the **first TLS connection that names the host**
(SNI); a client connecting by IP address gets none.

{{< callout type="info" >}}
**ALPN**

With ACME, the server's ALPN list holds `acme-tls/1`, and a Go server
refuses a client whose list shares nothing with its own. So `http/1.1` is
**appended** for WebDAV and S3 — every browser and WebDAV client offers it —
and `sunrpc` for NFS, as RFC 9289 §5.2 identifies RPC-with-TLS. Not `h2`:
HTTP/2 is not offered over these listeners, and offering it would be a lie.
`acme-tls/1` stays on every listener, so tls-alpn-01 is answered wherever
the CA reaches.
{{< /callout >}}

## WebDAV is not served in the clear

{{< callout type="error" >}}
**BREAKING in v0.9.0: WebDAV with passwords on a reachable address is refused**

HTTP Basic is the password, base64'd, on every request, and a bearer token
is as good as one. So a configuration serving WebDAV **without TLS** on an
address other machines can reach, while anybody authenticates, is
**refused, not warned about** — a warning scrolls by, and the password does
not come back:

```
webdav on 0.0.0.0:8080 would carry passwords in the clear: serve it with
tls = true, or -- when TLS is terminated in front of it, by a proxy -- say
plaintext = true
```
{{< /callout >}}

A configuration that started before v0.9.0 with
`serve "webdav" { addr = "0.0.0.0:8080" }` does not start after it. Two ways
out:

```hcl
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true            # with a tls block
}
```

or, when a reverse proxy terminates TLS in front of it and forwards in the clear:

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true       # TLS is terminated in front of this, on purpose
}
```

**Loopback** addresses (`127.0.0.1`, `::1`, `localhost`) and a server with
**nobody to authenticate** — no `user`, no `users`, no `oidc` block — are
unaffected. A `serve` block with no `addr` lands on loopback, and is unaffected
too.

`plaintext` is for WebDAV only, the protocol that sends a password, and
`plaintext` together with `tls` is refused: one or the other.

## What `check` says

[`fileshare check`]({{< relref "/configuration/check.md" >}}) prints, per protocol, what is
encrypted and what is in the clear **on purpose**:

```
webdav: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem
nfs: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem; clients present one signed by /etc/fileshare/tls/clients.pem (the machine, not the person)
```

and, for WebDAV behind a proxy,

```
webdav: in the clear on 10.0.0.5:8080, on purpose (plaintext = true): TLS must be terminated in front of it
```

## Per protocol

| protocol | `tls = true` | |
|---|---|---|
| WebDAV | yes | HTTPS |
| S3 | yes | HTTPS |
| NFS | yes | RFC 9289; see below |
| SMB | **refused** | SMB 3 encrypts with its own keys |
| SFTP | **refused** | SFTP is SSH |

## NFS over TLS proves the machine, not the person

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

`client_ca_file` makes a client present a certificate from that authority —
**which host is mounting** — and RFC 9289 leaves user authentication as it was:
the uid inside is still the one `AUTH_SYS` claims. So a share that names who may
use it is **still refused over NFS** without a `kerberos` block — or without
[identities from certificates]({{< relref "/security/nfs-certificates.md" >}}), which is a different use of
the same certificate.

TLS is **offered, not required**: a client that never asks for it is still
served the open shares. `client_ca_file` without `tls = true` is refused — it
would check nothing — and `client_ca_file` on any protocol other than NFS is
refused too.

## What else is refused

- `tls = true` with no `tls` block: nothing says where the certificate comes from.
- A `tls` block that no `serve` block uses: nothing would use it.
- An `http_challenge` that is not an address to listen on.
