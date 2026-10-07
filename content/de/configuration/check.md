---
title: "check, bevor Sie etwas neu starten, das in Benutzung ist"
linkTitle: "check, vor dem Neustart"
weight: 30
description: "Was fileshare check über eine Konfiguration ausgibt, bevor sie bereitgestellt wird, und wie man es liest."
tags: [konfiguration, check]
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

Jedes Image wird so geöffnet, wie der Server es öffnet, und wieder geschlossen; nichts wird
bereitgestellt. Das heißt **lesend und schreibend** für eine Freigabe, die nicht `read_only = true` sagt
– nur ein Gerät wird immer schreibgeschützt und exklusiv geöffnet, sodass ein Gerät, das der
laufende Server hält, hier als belegt abgelehnt werden kann.

Es gibt aus, **woher Zugangsdaten kommen, und nie, was sie enthalten** – und die
Spalten pro Person sind der Grund, warum es
[`Identity.Can`](https://github.com/go-authn/directory) gibt: eli ist in
derselben Gruppe wie dora, und SMB kann ihn trotzdem nicht bedienen, weil LDAP sein
Passwort hält und es niemandem gibt.

## Die Spalten lesen {#reading-the-columns}

| | |
|---|---|
| ein Dateisystem mit Sternchen | der Treiber wurde von der Freigabe **festgelegt**, nicht erkannt – siehe [Freigaben]({{< relref "/configuration/shares.md" >}}) |
| `NO` unter einem Protokoll | diese Freigabe wird dort nicht exportiert, und der Grund steht unter der Tabelle |
| `-` unter einem Protokoll, pro Freigabe | `protocols` der Freigabe lässt dieses weg |
| `-` unter einem Protokoll, pro Benutzer | die Quelle dieser Person kann nicht beweisen, was das Protokoll braucht – siehe [Identität]({{< relref "/configuration/identity.md" >}}) |

## Was es außerdem sagt {#what-else-it-says}

Nach den Tabellen sagt `check`, was man wissen muss, bevor man sich auf
die Konfiguration verlässt, sofern es zutrifft:

- bei einem `oidc`-Block, dass seine Tokens nur über WebDAV akzeptiert werden, und
  ob die Namen des Anbieters lokale Namen sind (`local_names`) – siehe
  [Identität]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}});
- pro Protokoll, was **verschlüsselt** ist und was **absichtlich im Klartext**
  – siehe [TLS]({{< relref "/security/tls.md#what-check-says" >}});
- woher eine **Sperrliste** kommt und was geschieht, solange sie nicht
  abgerufen werden kann – siehe [was was widerruft]({{< relref "/security/_index.md" >}});
- welche Freigaben über `DisableShare` der Admin-API **offline genommen** wurden
  und nicht bereitgestellt werden – siehe [die Admin-API]({{< relref "/administration/_index.md#disabling-a-share" >}}).

Bei einem Server mit einer KRL, NFS-Identitäten aus Zertifikaten und Shared Signals
lautet dieser Teil:

```
sftp: the provider's SSH certificates are checked against the KRL at https://bridge.example.org/ssh/krl, at login and on every operation; while it is unknown or older than 1h0m0s they are refused

federated people: revocations come from the shared signals transmitter https://bridge.example.org (CAEP session-revoked, polled); whatever the provider issued them before a revocation is refused -- tokens, SSH and OpenPubkey certificates, NFS certificates -- and open sessions stop. Refused while it is not heard from within 10m0s; revocations kept 192h0m0s in /var/lib/fileshare/revocations.json

nfs: identities come from client certificates signed by /etc/fileshare/bridge-x509-ca.pem, revoked by the CRL at https://bridge.example.org/x509/crl (refused while it is unknown or older than 1h0m0s).
     ⛔ a Linux client's certificate belongs to a MOUNT: every user of that mount is the person it names
```

`check` ruft die Listen nicht ab: Es sagt, woher sie kommen und was ihr
Fehlen bewirken wird. Das Discovery-Dokument und die Schlüssel des Identitätsanbieters
werden dagegen **sehr wohl** gelesen – ein `oidc`-Block, dessen Aussteller nicht erreichbar ist,
wird von `check` abgelehnt.
