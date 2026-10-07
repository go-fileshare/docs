---
title: "WebDAV — Basic, ou un jeton porteur"
linkTitle: "WebDAV : Basic, ou un jeton porteur"
weight: 20
description: "WebDAV avec HTTP Basic ou un jeton porteur OIDC, le TLS qu'il exige, et comment un jeton devient ici une personne."
tags: [protocoles, webdav, oidc]
---

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

HTTP Basic, ou un jeton porteur (bearer) — sur [TLS]({{< relref "/security/tls.md" >}}), depuis des fichiers ou
par ACME.

{{< callout type="error" >}}
**Pas en clair, depuis la v0.9.0**

HTTP Basic, c'est le mot de passe à chaque requête, et un jeton porteur vaut
un mot de passe. WebDAV **sans TLS**, sur une adresse que d'autres machines peuvent atteindre,
avec des personnes à authentifier, est **refusé au démarrage** —
`serve "webdav" { addr = "0.0.0.0:8080" }`, l'exemple que montrait autrefois cette page,
ne démarre plus. Quand un proxy termine TLS devant lui, dites-le :

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true
}
```

La boucle locale, et un serveur sans personne à authentifier, ne sont pas concernés. Voir
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

{{< callout type="info" >}}
**404, pas 403**

Un partage qu'une personne ne peut pas utiliser répond **404**. Son existence n'est pas confirmée.
{{< /callout >}}

## Un jeton, sur le seul protocole qui puisse en porter un {#a-token-over-the-one-protocol-that-can-carry-one}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
}
```

Un navigateur a un jeton et pas de mot de passe. WebDAV accepte donc `Authorization:
Bearer`, et là où il y a aussi des personnes locales, le défi qu'il envoie propose
**les deux** — un client choisit celui auquel il sait répondre. Le jeton est vérifié par
[go-authn/oidc](https://github.com/go-authn/oidc) : signature, émetteur, audience,
expiration.

{{< callout type="error" >}}
**Un jeton porteur : WebDAV seulement**

SMB s'authentifie avec NTLMv2, SFTP avec une clé ou un certificat, NFS avec
rien du tout, et S3 avec une signature SigV4 : aucun d'eux n'a d'endroit où
mettre un en-tête `Authorization`. C'est un fait propre aux protocoles, pas une
limite de ce programme.

La parole du fournisseur atteint **SFTP** plutôt dans un certificat — voir
[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}). Une configuration
qui nomme un fournisseur sans servir ni WebDAV ni un tel SFTP est **refusée
plutôt que démarrée**.
{{< /callout >}}

{{< callout type="error" >}}
**Un jeton dit qui le fournisseur pense qu'est quelqu'un. Il ne dit pas que ce serveur a un partage pour lui.**

Un jeton valide pour un nom qu'aucune source ici ne connaît est refusé — la lecture sûre
de *je ne vous connais pas* n'est pas *vous êtes autorisé*. Un nom qu'un compte local
porte aussi n'obtient pas non plus ce compte, sauf si `local_names = true` indique que les
noms du fournisseur sont ceux de ce serveur (désactivé par défaut, depuis la v0.20.0) ; sans
ce réglage, un tel jeton est refusé lui aussi, sauf si une règle `oidc:` nomme la personne.
{{< /callout >}}

Un site où le fournisseur **est** l'annuaire le dit :

```hcl
oidc {
  issuer    = "https://login.example.org"
  audience  = "fileshare"
  trust_all = true      # everybody that provider vouches for, not just these people
}
```

Il n'y a **aucun flux de connexion** ici : ni redirection, ni secret client, ni cookies.
C'est le serveur de ressources.

**Le nom** que donne un jeton est `preferred_username`, ou la revendication que nomme
`username_claim`. Sans `preferred_username`, c'est l'adresse électronique seulement si
le fournisseur dit `email_verified: true`, et `sub` sinon. Avec
`username_claim = "email"`, un jeton dont l'adresse n'est pas vérifiée est refusé —
en WebDAV, comme pour une connexion opkssh en SFTP. Tant qu'elle n'est pas vérifiée,
l'adresse n'est que ce que la personne a saisi, et un jeton signé qui la porterait
lierait une adresse que personne n'a vérifiée (OpenID Connect Core 5.1 ;
go-authn/oidc v0.2.0, fileshare v0.16.3).

Quelles personnes du fournisseur obtiennent un partage — des groupes, des personnes nommées, des
établissements entiers — s'écrit avec des règles `oidc:` et `domains` ; voir
[les personnes que nomme le fournisseur d'identité]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).

## Un jeton survit à la personne, sauf si quelque chose dit le contraire {#a-token-outlives-the-person-unless-something-says-otherwise}

Un jeton est vérifié ici, de façon autonome, et reste valide jusqu'à son expiration — quoi
qu'il soit arrivé à la personne depuis. Aucune liste de révocation ne peut le nommer. Un
[bloc `ssf`]({{< relref "/security/shared-signals.md" >}}) fait qu'un `session-revoked` CAEP
venu du fournisseur annule tout jeton émis (`iat`) avant lui.
