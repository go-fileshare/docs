---
title: "Ce qui révoque quoi"
linkTitle: "Sécurité"
weight: 40
description: "Quel mécanisme reprend chaque type d'identifiant qu'accepte fileshare, et pourquoi chacun d'eux refuse en cas de doute."
tags: [sécurité, révocation]
---

Un identifiant est vérifié au moment où il est présenté. Retirer ensuite l'accès à une personne —
un membre du personnel parti, un compte désactivé dans go-authn/bridge, un IdP entier
coupé — doit atteindre **chaque** identifiant qui lui a été donné, ainsi que les
sessions que ceux-ci ont déjà ouvertes. Chaque type d'identifiant est ici atteint par un
mécanisme différent, parce que chacun est délivré et vérifié différemment :

| identifiant | protocole | ce qui le reprend | ajouté en |
|---|---|---|---|
| un mot de passe, une empreinte NT ou un mot de passe d'application dans un annuaire `users` (SQL, LDAP) | SMB, WebDAV, S3 | la ligne supprimée, puis un [rechargement de l'annuaire]({{< relref "/administration/reload.md" >}}) : une nouvelle génération, et les connexions qu'il avait ouvertes fermées | v0.10.0 |
| une autorisation sur un partage | tous les protocoles | `Revoke` de l'[API d'administration]({{< relref "/administration/_index.md" >}}), ou `DisableShare` : une nouvelle génération, connexions fermées | v0.8.0 |
| un certificat SSH signé par l'AC du fournisseur | SFTP | sa [KRL]({{< relref "/security/revocation-lists.md" >}}), vérifiée à la connexion **et à chaque opération** ; et les [signaux partagés]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |
| un certificat OpenPubkey (opkssh) | SFTP | aucune liste ne peut le nommer ; les [signaux partagés]({{< relref "/security/shared-signals.md" >}}), et `opkssh_max_age` | v0.13.0 |
| un jeton d'accès | WebDAV | aucune liste ne peut le nommer ; les [signaux partagés]({{< relref "/security/shared-signals.md" >}}) | v0.13.0 |
| un certificat client X.509 qui nomme une personne | NFS | sa [CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}), à l'appel suivant ; et les [signaux partagés]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |

Sans le mécanisme de la troisième colonne, un identifiant reste valable jusqu'à son
expiration — et pour un certificat ou un jeton, cette fenêtre est sa durée de vie.

## Chacun d'eux refuse en cas de doute {#every-one-of-them-fails-closed}

Une liste de révocation qui ne peut pas être récupérée, une copie plus ancienne que son
`max_age`, une CRL qui a dépassé son propre `NextUpdate`, un émetteur de signaux partagés
resté muet au-delà de `max_age` : rien de tout cela n'est lu comme « rien n'est révoqué ». Cela
signifie « ce serveur ne sait pas », et les identifiants que gouverne ce mécanisme sont
**refusés** jusqu'à ce qu'il sache de nouveau.

{{< callout type="error" >}}
**Le prix est la disponibilité, et il est payé délibérément**

Une vérification de révocation qui laisse tout passer quand la liste est
injoignable est celle que déjoue un attaquant capable de bloquer la liste — la
raison pour laquelle l'OCSP en soft-fail des navigateurs a été comparé à « une ceinture de sécurité qui casse
au moment de l'accident ». La liste, ou l'émetteur, doit donc être servi aussi fiablement que les
connexions qu'il gouverne, et les métriques qui signalent qu'elle devient périmée sont celles
sur lesquelles [déclencher une alerte]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

Une copie n'est remplacée que par une liste qui a pu être analysée : une liste qui arrive cassée laisse
en place la dernière bonne, qui continue de compter vers `max_age` à partir du moment où **elle**
a été récupérée. Et une liste n'est récupérée qu'**en HTTPS** — une liste qu'un attaquant
placé sur le chemin peut remplacer ne révoque rien — si bien qu'une URL en HTTP simple est refusée au
démarrage.

## Transport {#transport}

Les identifiants eux-mêmes ne doivent pas traverser un réseau en clair. C'est le rôle de
[TLS]({{< relref "/security/tls.md" >}}) : WebDAV, S3 et NFS sur TLS, un certificat depuis des fichiers ou par ACME,
et — depuis la v0.9.0 — **WebDAV avec mots de passe refusé en clair**.
