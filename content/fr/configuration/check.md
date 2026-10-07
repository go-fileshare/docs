---
title: "check, avant de redémarrer ce que des gens utilisent"
linkTitle: "check, avant de redémarrer"
weight: 30
description: "Ce que fileshare check affiche sur une configuration avant qu'elle ne soit servie, et comment le lire."
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

Chaque image est ouverte comme le serveur l'ouvre, puis refermée ; rien n'est
servi. Cela signifie **en lecture-écriture** pour un partage qui ne dit pas `read_only = true`
— seul un périphérique est toujours ouvert en lecture seule, et en exclusivité : un périphérique que
détient le serveur en cours d'exécution peut donc être refusé ici comme occupé.

Il affiche **d'où vient un identifiant, jamais ce qu'il contient** — et les
colonnes par personne sont la raison d'être de
[`Identity.Can`](https://github.com/go-authn/directory) : eli est dans le
même groupe que dora et SMB ne peut toujours pas le servir, parce que LDAP détient son
mot de passe et ne le donne à personne.

## Lire les colonnes {#reading-the-columns}

| | |
|---|---|
| un système de fichiers suivi d'un astérisque | le pilote a été **affirmé** par le partage, pas reconnu — voir [partages]({{< relref "/configuration/shares.md" >}}) |
| `NO` sous un protocole | ce partage n'y est pas exporté, et la raison est affichée sous le tableau |
| `-` sous un protocole, par partage | les `protocols` du partage excluent celui-ci |
| `-` sous un protocole, par utilisateur | la source de cette personne ne peut pas prouver ce dont le protocole a besoin — voir [identité]({{< relref "/configuration/identity.md" >}}) |

## Ce qu'il dit d'autre {#what-else-it-says}

Après les tableaux, `check` dit ce qu'une personne doit savoir avant de se fier à
la configuration, quand c'est pertinent :

- avec un bloc `oidc`, que ses jetons ne sont acceptés qu'en WebDAV, et
  si les noms du fournisseur sont des noms locaux (`local_names`) — voir
  [identité]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}) ;
- par protocole, ce qui est **chiffré** et ce qui est **en clair délibérément**
  — voir [TLS]({{< relref "/security/tls.md#what-check-says" >}}) ;
- d'où vient une **liste de révocation**, et ce qui se passe tant qu'elle ne peut pas être
  récupérée — voir [ce qui révoque quoi]({{< relref "/security/_index.md" >}}) ;
- quels partages ont été **mis hors ligne** par le `DisableShare` de l'API d'administration,
  et ne sont pas servis — voir [l'API d'administration]({{< relref "/administration/_index.md#disabling-a-share" >}}).

Pour un serveur avec une KRL, des identités NFS issues de certificats et des signaux partagés,
cette partie se lit ainsi :

```
sftp: the provider's SSH certificates are checked against the KRL at https://bridge.example.org/ssh/krl, at login and on every operation; while it is unknown or older than 1h0m0s they are refused

federated people: revocations come from the shared signals transmitter https://bridge.example.org (CAEP session-revoked, polled); whatever the provider issued them before a revocation is refused -- tokens, SSH and OpenPubkey certificates, NFS certificates -- and open sessions stop. Refused while it is not heard from within 10m0s; revocations kept 192h0m0s in /var/lib/fileshare/revocations.json

nfs: identities come from client certificates signed by /etc/fileshare/bridge-x509-ca.pem, revoked by the CRL at https://bridge.example.org/x509/crl (refused while it is unknown or older than 1h0m0s).
     ⛔ a Linux client's certificate belongs to a MOUNT: every user of that mount is the person it names
```

`check` ne récupère pas les listes : il dit d'où elles viennent et ce que leur
absence provoquera. Le document de découverte et les clés du fournisseur d'identité, en
revanche, **sont** lus — un bloc `oidc` dont l'émetteur est injoignable est
refusé par `check`.
