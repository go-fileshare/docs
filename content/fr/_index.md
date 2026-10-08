---
title: "go-fileshare"
linkTitle: "Accueil"
type: docs
cascade:
  type: docs
description: "go-fileshare sert des images disque et des répertoires en SMB, NFS, WebDAV, SFTP et S3, avec un seul ensemble d'utilisateurs et des règles par partage, depuis une seule configuration."
---

**Une image disque — ou un répertoire — servie en SMB, NFS, WebDAV, SFTP et S3 :
les mêmes utilisateurs, le même accès par partage, depuis un seul fichier de configuration.** Pur
Go, `CGO_ENABLED=0`, un seul binaire. Ces pages décrivent la
[v0.25.0]({{< relref "/status.md" >}}).

```sh
go install github.com/go-fileshare/fileshare@latest

fileshare --image disk.img --user alice --password-file pw   # one image, now
fileshare --config /etc/fileshare.d                          # several, with users
```

Le mot de passe vient d'un **fichier**, jamais d'une option : un argument est visible
dans la liste des processus par tous les utilisateurs de la machine.

## Pourquoi un seul programme {#why-one-program}

Le protocole qui transporte une image dépend du client à l'autre bout :
macOS et Windows se tournent vers SMB, un parc Linux a déjà NFS, un navigateur ou un
téléphone a HTTP. Faire tourner trois serveurs, chacun avec son propre fichier de configuration
et sa propre idée de qui est `alice`, c'est s'exposer à ce que deux d'entre eux se trompent subtilement.

Il y a donc une seule configuration, un seul ensemble d'utilisateurs, un seul ensemble de règles par partage —
et les protocoles sont des écouteurs posés dessus.

## Ce qu'il affiche au démarrage {#what-it-prints-at-startup}

```
smb    on 0.0.0.0:445 — photos and scratch
webdav on 0.0.0.0:8080 — photos and scratch
sftp   on 0.0.0.0:2222 — photos and scratch
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

Ce dernier paragraphe donne la forme de tout le programme : une règle que la configuration
demande et un protocole qui ne peut pas l'honorer ne se rejoignent pas discrètement à mi-chemin.
Voir [Ce qu'un protocole peut promettre]({{< relref "/protocols/_index.md" >}}).

## Pour aller plus loin {#where-to-go-next}

| | |
|---|---|
| [La configuration]({{< relref "/configuration/_index.md" >}}) | utilisateurs, groupes, partages, blocs `serve` |
| [Partages : images, périphériques, répertoires]({{< relref "/configuration/shares.md" >}}) | ce qui est détecté, ce qui doit être déclaré |
| [Utilisateurs, groupes et annuaires]({{< relref "/configuration/identity.md" >}}) | fichiers, SQL, LDAP, et ce que chacun peut prouver |
| [`check`]({{< relref "/configuration/check.md" >}}) | relire toute la configuration avant de redémarrer |
| [Protocoles]({{< relref "/protocols/_index.md" >}}) | ce que chacun peut et ne peut pas promettre |
| [L'API d'administration]({{< relref "/administration/_index.md" >}}) | des partages créés, accordés et mis hors ligne sans redémarrage |
| [Volumes]({{< relref "/administration/volumes.md" >}}) | jeux de données ZFS, sous-volumes btrfs et quotas de projet XFS/ext4, créés par l'API et servis |
| [Santé et métriques]({{< relref "/administration/health.md" >}}) | `/healthz`, `/readyz`, `/metrics` |
| [Relire l'annuaire]({{< relref "/administration/reload.md" >}}) | `reload`, `SIGHUP`, et ce qu'une suppression fait aux sessions ouvertes |
| [TLS]({{< relref "/security/tls.md" >}}) | fichiers ou ACME ; et pourquoi WebDAV est refusé en clair |
| [Ce qui révoque quoi]({{< relref "/security/_index.md" >}}) | KRL, CRL, signaux partagés — et pourquoi chacun refuse en cas de doute |
| [Ne compiler que ce que l'on veut]({{< relref "/operations/build-tags.md" >}}) | une étiquette laisse un protocole entièrement de côté |
| [Un processus par protocole]({{< relref "/operations/isolation.md" >}}) | `--isolate` |
| [État]({{< relref "/status.md" >}}) | ce qui est vérifié, et ce qui n'est pas encore là |

## Licence {#licence}

BSD-3-Clause.
