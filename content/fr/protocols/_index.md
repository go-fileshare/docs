---
title: "Ce qu'un protocole peut promettre"
linkTitle: "Protocoles"
weight: 20
description: "Pourquoi chaque protocole peut ou ne peut pas savoir qui demande, et ce que cela implique pour les partages qu'il sert."
tags: [protocoles]
---

Les protocoles ne s'accordent pas sur la seule chose dont le contrôle d'accès a besoin : savoir si
le serveur peut dire **qui** demande.

| | |
|---|---|
| **[SMB]({{< relref "/protocols/smb.md" >}})** | NTLMv2. Le mot de passe ne traverse jamais le réseau, et le partage dit à un lecteur qu'il n'est que lecteur — dans le masque d'accès, avant qu'il n'essaie. |
| **[WebDAV]({{< relref "/protocols/webdav.md" >}})** | HTTP Basic, sur [TLS]({{< relref "/security/tls.md" >}}) : refusé en clair sur une adresse joignable, sauf si `plaintext = true` indique qu'un proxy termine TLS devant lui. Un partage qu'une personne ne peut pas utiliser répond 404, pas 403 : son existence n'est pas confirmée. |
| **[SFTP]({{< relref "/protocols/sftp.md" >}})** | Une **clé publique**, ou un **certificat SSH** émis par une autorité de confiance : le serveur ne détient jamais le secret, et avec un certificat, l'accès d'une personne est délivré et expire ailleurs. Pas de mot de passe : un client qui en demande un fait précisément ce que les clés existent pour éviter. |
| **[OIDC]({{< relref "/protocols/webdav.md" >}})** (sur WebDAV) | Un **jeton porteur** (bearer) signé par un fournisseur d'identité. Vérifié par [go-authn/oidc](https://github.com/go-authn/oidc) : signature, émetteur, audience, expiration. Aucun autre protocole ici n'a d'endroit où en mettre un — S3 signe avec SigV4, qui n'a pas de champ pour un jeton porteur. En [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}), la parole du fournisseur arrive plutôt dans un certificat. |
| **[S3]({{< relref "/protocols/s3.md" >}})** | **SigV4**, en en-tête ou présigné. Le secret se prouve en calculant un HMAC et ne traverse jamais le réseau — si bien que, comme pour NTLMv2, l'annuaire doit DÉTENIR le mot de passe et pas seulement le vérifier. Un partage est un bucket. |
| **[NFSv3]({{< relref "/protocols/nfs.md" >}})** | **Rien**, à lui seul. `AUTH_UNIX` est une affirmation — le client dit « uid 501 » et le réseau ne peut pas le contredire. Un bloc `kerberos` lève cette limite, tout comme les [identités issues de certificats]({{< relref "/security/nfs-certificates.md" >}}). [TLS]({{< relref "/security/tls.md" >}}) le chiffre, et prouve la machine, pas la personne. |

## La conséquence, énoncée une fois {#the-consequence-stated-once}

**Un partage qui nomme qui peut l'utiliser n'est pas exporté en NFS** — sauf si
Kerberos est configuré, ou si NFS tire les
[identités de certificats]({{< relref "/security/nfs-certificates.md" >}}).

Ni un avertissement, ni une option : une configuration qui dit *photos appartient à alice*
et un protocole qui remet photos à quiconque se connecte ne peuvent pas être honorés tous les deux, et
élargir discrètement l'accès est le pire des deux échecs. Le refus est
affiché au démarrage et dans [`check`]({{< relref "/configuration/check.md" >}}), avec sa raison.

## Ce que chacun attend d'un annuaire {#what-each-one-needs-from-a-directory}

Les protocoles qui peuvent servir une personne donnée dépendent de ce que sa source peut
prouver, et pas de la seule configuration. Voir
[ce qu'une source peut prouver]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}).

## Ce qui est chiffré {#what-is-encrypted}

WebDAV, S3 et NFS acceptent `tls = true` ; SMB 3 chiffre avec ses propres clés, et SFTP
est du SSH. Voir [TLS, et certificats par ACME]({{< relref "/security/tls.md" >}}).
