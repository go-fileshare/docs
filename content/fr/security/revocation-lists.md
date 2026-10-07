---
title: "Révoquer des certificats SSH : la KRL"
weight: 20
description: "Révoquer les certificats SSH du fournisseur d'identité avec une KRL OpenSSH signée et datée, vérifiée à la connexion et à chaque opération SFTP."
tags: [sécurité, révocation, sftp, ssh]
---

Un certificat signé par l'AC SSH du fournisseur d'identité — voir
[SFTP pour les personnes dont le fournisseur se porte garant]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})
— est vérifié à la connexion. Désactiver quelqu'un dans go-authn/bridge révoque ses
jetons et supprime ses mots de passe d'application, ce qu'un
[rechargement de l'annuaire]({{< relref "/administration/reload.md" >}}) applique ici ; mais sans
liste, un certificat déjà émis ouvrirait encore SFTP jusqu'à son expiration, et
une session SFTP déjà ouverte le resterait.

Depuis la v0.12.0, une liste ferme cette porte :

```hcl
oidc {
  issuer          = "https://login.example.org"
  audience        = "fileshare"
  ssh_ca_file     = "/etc/fileshare/bridge-ca.pub"
  ssh_krl_url        = "https://bridge.example.org/ssh/krl"   # or ssh_krl_file
  ssh_krl_state_file = "/var/lib/fileshare/ssh.krl.state"    # the order, across a restart
  ssh_krl_max_age    = "1h"                                  # default; ssh_krl_refresh = "1m"
}
```

go-authn/bridge publie les certificats SSH qu'il a révoqués — une personne
désactivée, un IdP désactivé — sous forme de **KRL** OpenSSH (`PROTOCOL.krl`, la liste
que lit `RevokedKeys` de `sshd`), et fileshare en garde une copie, lue avec
[go-authn/krl](https://github.com/go-authn/krl).

## Refusé à la connexion, et au milieu d'une session {#refused-at-login-and-in-the-middle-of-a-session}

Un certificat révoqué est refusé à la connexion, et **une session qu'il a déjà ouverte
cesse d'être servie**. SSH vérifie un certificat une seule fois, lors de la poignée de main ; chaque
opération d'une session SFTP fédérée interroge donc à nouveau la KRL, **fichiers ouverts
compris**. La première opération qui suit la modification de la copie de la liste détenue par ce serveur est refusée.

## Champs {#fields}

| champ | par défaut | |
|---|---|---|
| `ssh_krl_url` | | où la liste est récupérée, **HTTPS uniquement** |
| `ssh_krl_file` | | ou un fichier, par chemin absolu — l'un ou l'autre, jamais les deux |
| `ssh_krl_ca_file` | celles du système | épingle les autorités face auxquelles le serveur HTTPS de la liste est vérifié |
| `ssh_krl_refresh` | `1m` | la fréquence de récupération, au moins une seconde |
| `ssh_krl_max_age` | `1h` | l'âge maximal de la dernière bonne copie avant que les certificats qu'elle gouverne ne soient refusés |
| `ssh_krl_state_file` | | où est conservée la dernière liste vérifiée, pour qu'un redémarrage se souvienne de l'ordre |

Un `max_age` plus court que `refresh` est refusé — chaque copie serait périmée avant
la récupération suivante. Une KRL sans `ssh_ca_file` est refusée aussi : elle révoque les
certificats du fournisseur, et sans cette AC il n'y en a aucun.

## Elle refuse en cas de doute {#it-fails-closed}

{{< callout type="error" >}}
**Tant que la KRL ne peut pas être récupérée, les certificats du fournisseur sont refusés**

Tant que la liste ne peut pas être récupérée, ou que sa dernière bonne copie est plus ancienne que
`ssh_krl_max_age`, **tous** les certificats signés par l'AC du fournisseur sont
refusés : une vérification de révocation qui laisse tout passer quand la liste est
injoignable est celle que déjoue un attaquant capable de bloquer la liste. La liste
doit alors être servie aussi fiablement que les connexions qu'elle gouverne.
`fileshare_revocation_list_age_seconds{list="ssh_krl"}` est la métrique sur laquelle
alerter avant d'atteindre `max_age` ; voir
[santé et métriques]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

## Signée et datée, depuis `ssh_krl_url` {#signed-and-dated-from-ssh_krl_url}

Depuis la v0.15.0, **une KRL obtenue par `ssh_krl_url` doit être signée et dire quand elle
expire**, comme le décrit [go-authn/revocation](https://github.com/go-authn/revocation)
et comme go-authn/bridge la sert depuis la v0.10.0 :

- **La signature :** `<url>.sig` est une signature SSHSIG détachée de l'AC SSH du
  fournisseur (`ssh_ca_file`, que `ssh_krl_url` exige donc), dans
  l'espace de noms `krl@go-authn.github.io`. Elle est récupérée avec `If-Match` sur
  l'ETag de la liste, et récupérée de nouveau aussitôt si la liste a été réémise entre-temps.
- **L'expiration :** l'extension `expires@go-authn.github.io`. Passé ce terme, la liste
  n'est plus à jour, si récemment qu'un `304` l'ait confirmée.

HTTPS (`ssh_krl_ca_file` épingle ses autorités) authentifie le serveur qui
a répondu, pas la liste : un miroir, un cache ou un serveur web compromis pourrait
en servir une vide, et rien dans une KRL seule ne dit qu'elle est périmée. Une liste qui
ne se vérifie pas est refusée, et la dernière bonne copie est conservée. Une liste déposée à
la main avec `ssh_krl_file` a la confiance qu'on accorde au système de fichiers qui la contient, et n'a besoin
ni de signature ni d'expiration ; celles qu'elle porte sont honorées.

{{< callout type="warning" >}}
**Passer à la v0.15.0**

Un `ssh_krl_url` qui sert une KRL non signée, ou sans expiration, n'est
plus accepté : les certificats du fournisseur sont alors refusés, comme pour toute
liste impossible à obtenir. Déployez d'abord go-authn/bridge v0.10.0 ou ultérieure.
{{< /callout >}}

## Jamais en arrière, même après un redémarrage {#never-backwards-across-a-restart-too}

Depuis la v0.16.0, une liste est refusée si elle est plus ancienne que celle détenue : une
version inférieure (la version de la KRL, le numéro d'une CRL), ou la même version émise
plus tôt. Avec `ssh_krl_state_file` (`crl_state_file` pour
[NFS]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})), la dernière liste vérifiée est
conservée sur disque, vérifiée de nouveau au démarrage, et ordonne la suivante. À elle seule, elle
ne compte pas comme à jour tant qu'une liste n'a pas été récupérée. Sans ce fichier, un redémarrage
oublie l'ordre, et fileshare le dit au démarrage : une liste plus ancienne, encore signée
et non expirée, serait acceptée.

Ces fichiers, comme `ssh_ca_file`, les fichiers d'AC, les fichiers de listes et (depuis la
v0.17.0) `authorized_keys_file`, `dsn_file`, `bind_password_file` et le
`ca_file` de `ssf`, **ne peuvent pas se trouver dans un partage** : quiconque y écrit
déciderait de qui entre.

## Un `sshd` ordinaire à côté de fileshare {#a-plain-sshd-next-to-fileshare}

La même vérification est disponible sans fileshare : `revokd`, de go-authn/revocation,
récupère les listes, ne conserve que celles qui se vérifient, garde leur ordre sur
disque, écrit `RevokedKeys` pour `sshd` et, quand une liste expire, en écrit une qui
révoque l'AC elle-même, si bien que `sshd` refuse lui aussi en cas de doute.

## Ce qu'elle ne couvre pas {#what-it-does-not-cover}

**Les certificats OpenPubkey (opkssh) ne figurent dans aucune KRL** — rien ne les a émis
sinon la propre clé de la personne. Ils sont bornés par `opkssh_max_age`, et repris
par les [signaux partagés]({{< relref "/security/shared-signals.md" >}}). De même qu'un jeton d'accès en WebDAV.

Le même mécanisme — récupération, conservation, refus en cas de doute — porte la **CRL X.509**
des [identités NFS issues de certificats]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}).
