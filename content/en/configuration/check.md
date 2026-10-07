---
title: "check, before you restart something people are using"
linkTitle: "check, before you restart"
weight: 30
description: "What fileshare check prints about a configuration before it is served, and how to read it."
tags: [configuration, check]
---

```
$ fileshare check /etc/fileshare.d
ATTIC

SHARE    IMAGE             FILESYSTEM  WHO MAY CONNECT           WHO MAY WRITE             SMB  WEBDAV  NFS  SFTP
photos   /srv/photos.img   fat32       alice and bob             alice                     yes  yes     NO   yes
scratch  /srv/scratch.img  ext4        anyone who authenticates  anyone who authenticates  yes  yes     yes  yes

photos is not served over nfs: it is restricted to alice and bob, and NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client makes about itself and the wire cannot disagree with it. A kerberos block lifts this: sec=krb5 carries a principal a ticket proves; so does identity = "certificate" on the nfs serve block: RPC-over-TLS with a client certificate naming the person

USER   FROM                      AUTHENTICATES WITH                         SMB  WEBDAV  SFTP
alice  the configuration file    a password and 1 key from /etc/…/alice.pw  yes  yes     yes
bob    the configuration file    a password from /etc/…/bob.pw              yes  yes     -
dora   ldaps://ldap.example.org  a password check and an NT hash            yes  yes     -
eli    ldaps://ldap.example.org  a password check                           -    yes     -

this configuration can be served
```

Every image is opened the way the server opens it and closed again; nothing is
served. That is **read-write** for a share that does not say `read_only = true`
— only a device is always opened read-only, and exclusively, so a device the
running server holds can be refused here as busy.

It prints **where a credential comes from and never what is in it** — and the
per-person columns are why
[`Identity.Can`](https://github.com/go-authn/directory) exists: eli is in the
same group as dora and SMB still cannot serve him, because LDAP holds his
password and gives it to nobody.

## Reading the columns

| | |
|---|---|
| a filesystem with an asterisk | the driver was **asserted** by the share, not recognised — see [shares]({{< relref "/configuration/shares.md" >}}) |
| `NO` under a protocol | that share is not exported there, and the reason is printed below the table |
| `-` under a protocol, per share | the share's `protocols` leaves that one out |
| `-` under a protocol, per user | that person's source cannot prove what the protocol needs — see [identity]({{< relref "/configuration/identity.md" >}}) |

## What else it says

After the tables, `check` says what a person needs to know before relying on
the configuration, when it applies:

- with an `oidc` block, that its tokens are accepted over WebDAV only, and
  whether the provider's names are local names (`local_names`) — see
  [identity]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}});
- per protocol, what is **encrypted** and what is **in the clear on purpose**
  — see [TLS]({{< relref "/security/tls.md#what-check-says" >}});
- where a **revocation list** comes from, and what happens while it cannot be
  fetched — see [what revokes what]({{< relref "/security/_index.md" >}});
- which shares were **taken offline** through the admin API's `DisableShare`,
  and are not served — see [the admin API]({{< relref "/administration/_index.md#disabling-a-share" >}}).

For a server with a KRL, NFS identities from certificates and shared signals,
that part reads:

```
sftp: the provider's SSH certificates are checked against the KRL at https://bridge.example.org/ssh/krl, at login and on every operation; while it is unknown or older than 1h0m0s they are refused

federated people: revocations come from the shared signals transmitter https://bridge.example.org (CAEP session-revoked, polled); whatever the provider issued them before a revocation is refused -- tokens, SSH and OpenPubkey certificates, NFS certificates -- and open sessions stop. Refused while it is not heard from within 10m0s; revocations kept 192h0m0s in /var/lib/fileshare/revocations.json

nfs: identities come from client certificates signed by /etc/fileshare/bridge-x509-ca.pem, revoked by the CRL at https://bridge.example.org/x509/crl (refused while it is unknown or older than 1h0m0s).
     ⛔ a Linux client's certificate belongs to a MOUNT: every user of that mount is the person it names
```

`check` does not fetch the lists: it says where they come from and what their
absence will do. The identity provider's discovery document and keys, on the
other hand, **are** read — an `oidc` block whose issuer cannot be reached is
refused by `check`.
