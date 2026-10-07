---
title: "Révoquer ce qu'aucune liste ne couvre : les signaux partagés"
linkTitle: "Signaux partagés"
weight: 40
description: "Révoquer des jetons d'accès, des certificats OpenPubkey et d'autres identifiants fédérés avec des événements CAEP session-revoked venus d'un émetteur de signaux partagés."
tags: [sécurité, révocation, signaux partagés]
---

Un certificat a une liste de révocation — la [KRL]({{< relref "/security/revocation-lists.md" >}}), la
[CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}). Un **jeton d'accès** que fileshare
vérifie de façon autonome, et un **certificat OpenPubkey** signé par la propre clé de la personne,
n'en ont pas : ils sont valables jusqu'à leur expiration, quoi qu'il soit arrivé à la
personne depuis.

Depuis la v0.13.0, l'[OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html)
(SSF, signaux partagés) referme cette brèche, avec un événement [CAEP](https://openid.net/specs/openid-caep-1_0-final.html)
**`session-revoked`** :

```hcl
ssf {
  transmitter        = "https://bridge.example.org"          # its issuer
  audience           = "https://files.example.org"           # what this server is to it
  client_id          = "fileshare"                           # OAuth client credentials,
  client_secret_file = "/etc/fileshare/ssf.secret"           # scope "ssf" (or token_file)
  state_file         = "/var/lib/fileshare/revocations.json"
  max_age            = "10m"                                 # default; retain = "192h"
}
```

go-authn/bridge en envoie un quand une personne ou un IdP est désactivé ; fileshare
l'**interroge** (polling, RFC 8936, le mode par défaut de SSF) et conserve, par personne, **quand**
c'est arrivé.

## Tout ce qui a été émis avant est refusé, sur tous les protocoles {#everything-issued-before-is-refused-on-every-protocol}

À partir de ce moment, tout identifiant fédéré **émis avant** est refusé —
et **les sessions que ceux-ci ont ouvertes cessent d'être servies**, par les mêmes contrôles que
ceux de la KRL. Le sens d'« émis » dépend de ce qui le porte :

| protocole | identifiant | sa date d'émission |
|---|---|---|
| WebDAV | un jeton d'accès | son `iat` |
| SFTP | un certificat signé par l'AC SSH du fournisseur | le début de sa validité (`ValidAfter`) |
| SFTP | un certificat OpenPubkey (opkssh) | l'`iat` du jeton d'identité |
| NFS | un certificat client ([identity = "certificate"]({{< relref "/security/nfs-certificates.md" >}})) | son `NotBefore` |

Ce que le fournisseur émet **après** lui appartient : une personne réactivée n'est pas
exclue.

{{< callout type="info" >}}
**La même seconde compte comme « avant »**

L'`event_timestamp` de CAEP et l'`iat` d'un jeton sont des secondes entières, et un
identifiant émis **au moment de la révocation ou avant** est refusé — si bien qu'un identifiant émis
dans la même seconde qu'une révocation est refusé lui aussi, et qu'une personne réactivée
juste après avoir été désactivée obtient des identifiants valides à partir de la seconde suivante.
C'est le côté prudent, délibérément. Une date d'émission inconnue
n'est pas « il y a longtemps » : elle est refusée dès qu'il existe une révocation pour cette
personne.
{{< /callout >}}

## De qui parle un événement {#who-an-event-is-about}

Le sujet est l'**`account`** de RFC 9493 (`acct:user@domain`, le nom qu'utilisent les
partages), **`iss_sub`**, **`email`**, ou des **`aliases`** de ceux-ci.

**Un IdP désactivé en entier** arrive sous forme de sujet *tenant* (locataire) CAEP, avec les
portées (scopes) de l'IdP dans l'événement : toute personne dont le nom est `@` l'une d'elles — comparée en entier,
`evil-univ-a.fr` n'est pas `univ-a.fr` — et dont l'identifiant a été émis avant
est refusée, **y compris les personnes dont le fournisseur ne se souvient plus**.

## Comment le récepteur s'authentifie {#how-the-receiver-authenticates}

Avec des **identifiants client OAuth** (client credentials, RFC 6749 §4.4, portée `ssf`) : `client_id` et
`client_secret_file`, le jeton étant récupéré auprès du propre point de terminaison de jetons de l'émetteur
— trouvé dans sa configuration OpenID, ou `token_url` — et renouvelé
avant son expiration. `token_file` existe à la place pour les émetteurs qui délivrent
un jeton porteur de longue durée. L'un ou l'autre : les deux, ou aucun, est refusé.

| champ | par défaut | |
|---|---|---|
| `transmitter` | | son émetteur (issuer), **HTTPS uniquement** |
| `audience` | | ce que ce serveur est pour l'émetteur ; obligatoire, sinon un SET adressé à un autre récepteur serait cru ici |
| `client_id`, `client_secret_file` | | identifiants client OAuth, ensemble |
| `token_url` | découvert | HTTPS uniquement, sinon le secret traverse le réseau en clair |
| `token_file` | | un jeton de longue durée au lieu d'identifiants client |
| `state_file` | | **obligatoire** : où les révocations sont consignées |
| `ca_file` | celles du système | épingle les autorités de l'émetteur |
| `max_age` | `10m` | combien de temps sans nouvelles de l'émetteur avant que les identifiants fédérés ne soient refusés ; au moins une minute |
| `retain` | `192h` | combien de temps une révocation est conservée ; au moins 169h (depuis la v0.14.0) |

Une révocation est **consignée avant d'être acquittée** : une révocation acquittée
survit donc à un redémarrage ; et elle est conservée pendant `retain` — plus longtemps que tout identifiant
qu'elle pourrait annuler (192h dépasse l'identifiant de plus longue durée qu'émet le fournisseur, un
certificat SSH de 168h).

Un bloc `ssf` sans rien de fédéré à révoquer — ni bloc `oidc`, ni NFS avec
`identity = "certificate"` — est refusé.

Le transport est [github.com/hstern/go-ssf](https://github.com/hstern/go-ssf) :
la découverte, l'interrogation RFC 8936, la couche JWS du SET. Ce que **signifie** un événement relève
de fileshare.

## Elle refuse en cas de doute {#it-fails-closed}

{{< callout type="error" >}}
**Tant que l'émetteur se tait, les identifiants fédérés sont refusés**

Comme pour la KRL : tant que l'émetteur n'a pas répondu à une interrogation dans le délai de
`max_age`, **tous les identifiants fédérés** sont refusés, parce que « aucune
révocation n'est arrivée » et « aucune ne pouvait arriver » se ressemblent.
`fileshare_ssf_last_heard_seconds` est la métrique sur laquelle alerter ;
`fileshare_ssf_revoked_subjects` indique combien de révocations sont conservées. Voir
[santé et métriques]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

## Vérifié de bout en bout {#checked-end-to-end}

La voie d'interopérabilité de go-authn/bridge
([#12](https://github.com/go-authn/bridge/pull/12),
[#17](https://github.com/go-authn/bridge/pull/17)) juge fileshare v0.13.0
face au vrai bridge : la KRL, la CRL en NFS, les jetons WebDAV révoqués par
SSF, un IdP désactivé par portée, et opkssh par SSF — le cas qu'aucune liste de révocation
n'atteint.
