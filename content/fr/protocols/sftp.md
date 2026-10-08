---
title: "SFTP — clés, ou certificats"
linkTitle: "SFTP : clés, ou certificats"
weight: 30
description: "SFTP avec des clés publiques ou des certificats SSH : autorités locales et du fournisseur, OpenPubkey, délégations de domaine, source-address, et révocation."
tags: [protocoles, sftp, ssh]
---

Tous les clients l'ont déjà : `sftp` est livré avec OpenSSH, le Finder et GNOME
le montent, les éditeurs le parlent. C'est aussi le seul protocole ici dont la forme
ne convient pas — une personne se connecte et arrive dans **un seul** système de fichiers, pas dans une liste de
partages.

Les partages deviennent donc les répertoires de premier niveau d'une arborescence construite pour la personne qui
vient de s'authentifier :

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

Un partage qu'alice ne peut pas utiliser **n'est pas un répertoire qu'alice peut voir**, et un partage qu'elle
ne peut que lire refuse ses écritures *dans l'arborescence*, avant un pilote qui les aurait
autorisées.

{{< callout type="info" >}}
**Un renommage entre deux partages est refusé**

Ce serait une copie et une suppression sur deux images, et ce n'est pas ce que
promet un renommage, où que ce soit.
{{< /callout >}}

## Configuration {#configuration}

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # optional

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # nothing here: her certificate is her credential
```

Avec `trusted_user_ca_file`, un certificat signé par cette autorité et qui nomme
l'utilisateur parmi ses principals suffit — l'accès d'une personne est donc **délivré et
expire ailleurs**, et aucun fichier ici n'est modifié quand quelqu'un arrive ou s'en va.

La signature, la période de validité et les principals sont vérifiés par le
`CertChecker` de `x/crypto/ssh` ; vérifié face au **propre client d'OpenSSH**,
qui refuse aussi la même clé dès que son certificat est mis de côté.

{{< callout type="warning" >}}
**Sans `host_key_file`, une nouvelle identité est générée à chaque démarrage**

Le serveur le dit, et chaque client qui l'a déjà vu avertira
d'un changement de clé — c'est le client qui fait son travail.
{{< /callout >}}

## Certificats destinés à cet hôte : la délégation de domaine {#certificates-meant-for-this-host-the-domain-grant}

Depuis la v0.22.0. L'[AC SSH de l'EuroHPC Federation Platform](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-overview/)
(GÉANT / MyAccessID) signe des certificats auxquels chaque site de la fédération
fait confiance. Chaque certificat indique à quels hôtes il est destiné, dans l'extension
`ssh-domain-grant@core.aai.geant.org`. sshd ignore cette extension,
et fileshare aussi jusqu'à la v0.22.0 : un certificat délégué pour les machines de quelqu'un
d'autre valait autant ici. Avec `ssh_domains`, la délégation est lue, analysée et
comparée par [go-authn/sshcert](https://github.com/go-authn/sshcert) comme le dit la
spécification. Un certificat dont la délégation ne nomme aucun des noms de cet hôte
est refusé.

```hcl
trusted_user_ca_file = "/etc/fileshare/efp-ssh-ca.pub"
ssh_domains          = ["files.example.org", "sftp.example.org"]
# ssh_accept_ungranted = true   # only if a trusted CA never writes a grant
```

| clé | |
|---|---|
| `ssh_domains` | les noms de cet hôte, tels qu'une délégation les nomme. Un certificat est accepté si l'un des motifs de sa délégation correspond à l'un d'eux. `*` vaut **un caractère ou plus au sein d'une seule étiquette** : `*.example.org` délègue `files.example.org` mais pas `a.files.example.org`, et `files-*.example.org` ne délègue pas `files-.example.org`. La comparaison ignore la casse. Ce sont des noms d'hôte, pas des motifs, chacun listé une seule fois, et au moins une autorité (`trusted_user_ca_file`, ou le `ssh_ca_file` du bloc `oidc`) doit être de confiance. |
| `ssh_accept_ungranted` | laisser entrer un certificat **sans** délégation. Désactivé par défaut ; sans `ssh_domains`, il est refusé comme dénué de sens. |

| certificat | `ssh_domains` | `+ ssh_accept_ungranted` | sans `ssh_domains` |
|---|---|---|---|
| délégué pour l'un des `ssh_domains` | accepté | accepté | accepté |
| délégué pour d'autres hôtes seulement, ou `[]` | refusé | refusé | accepté |
| sans délégation | refusé | accepté, journalisé | accepté |
| une délégation illisible (`null`, `[1]`, non compacte...) | refusé | **refusé** | accepté |

Sans `ssh_domains`, rien n'est lu et rien ne change. La délégation est lue sur
les certificats qu'une **autorité** a signés : ceux de `trusted_user_ca_file`, et ceux du
fournisseur (`ssh_ca_file`). Un certificat OpenPubkey est signé par la propre clé de
l'utilisateur : une délégation qu'il contiendrait serait la parole de l'utilisateur sur lui-même. Son audience est
plutôt `opkssh_client_id`. Les clés simples ne portent pas de délégation et ne sont pas concernées.

Chaque refus est journalisé avec le numéro de série du certificat, son key ID et la raison :

```
sftp: alice: certificate 42 ("alice@efp") refused: its domain grant [login.example.eu] names none of this host's names [files.example.org sftp.example.org]
sftp: alice: certificate 43 ("staff") refused: it has no domain grant (ssh-domain-grant@core.aai.geant.org), and ssh_domains requires one
```

Chaque décision est aussi comptée dans `fileshare_sftp_domain_grant_total{result}`,
avec le résultat `granted`, `accepted_ungranted`, `refused_not_granted`,
`refused_absent` ou `refused_malformed`. Un client propose un certificat avant de
prouver qu'il détient la clé : ces compteurs comptent donc des tentatives, pas des personnes.

{{< callout type="error" >}}
**Pourquoi refuser en cas de doute (fail closed)**

Prenez un hôte qui fait confiance à une seconde autorité : sa propre AC pour le personnel, une AC
de test, l'AC de préproduction d'EFP ajoutée au même fichier, ou un client go-authn/bridge
sans délégations. Les certificats de cette autorité ne portent pas de délégation. Si une délégation
absente signifiait « aucune restriction », le filtre ne filtrerait rien sur cet hôte.

Ici, un certificat sans délégation est refusé sauf si `ssh_accept_ungranted`
dit le contraire. Une délégation présente mais illisible est refusée,
quoi qu'elle dise : une autorité qui en a écrit une entendait restreindre le
certificat. go-authn/sshcert détaille le
[scénario à plusieurs AC](https://github.com/go-authn/sshcert#why-fail-closed-the-multi-ca-scenario).
{{< /callout >}}

### Avec l'AC d'EFP {#with-efps-ca}

`trusted_user_ca_file` est le `TrustedUserCAKeys` d'OpenSSH. Mettez-y la clé qu'EFP
publie à <https://sshca.my-eurohpc.eu/config>, comme l'indique la
[page de confiance](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-trust/) d'EFP.
Cette URL renvoie `{"PublicKey": "ssh-ed25519 ..."}` ; le fichier contient la valeur
de `PublicKey` sur une ligne à part :

```sh
curl -s https://sshca.my-eurohpc.eu/config | jq -r '.PublicKey' > /etc/fileshare/efp-ssh-ca.pub
```

Définissez ensuite `ssh_domains` avec les noms qu'utilise la délégation de votre entité d'hébergement. Le principal
unique d'un certificat EFP est l'identifiant MyAccessID
(`<id>@myaccessid.org`) : c'est donc le nom d'utilisateur sous lequel il se connecte. Un bloc `user`
de ce nom, sans identifiant propre, le reçoit :

```hcl
user "u1234@myaccessid.org" {}
```

### Avec go-authn/bridge {#with-go-authnbridge}

Un client [go-authn/bridge](https://github.com/go-authn/bridge) (v0.19.0 ou
ultérieure) avec `ssh_certificates = true` et
`ssh_domain_grants = ["files.example.org"]` émet des certificats délégués pour
cet hôte. Avec `ssh_principal_claim = "voperson_id"`, son principal unique est le
`voperson_id` de la personne, comme dans le profil EFP, et c'est le bloc `user` à
écrire ici. Faites confiance à l'AC de bridge comme ce serveur fait confiance à n'importe laquelle :

- en tant que `trusted_user_ca_file`, pour des comptes locaux. bridge sert sa clé au format
  d'EFP à `/ssh/config` : la ligne `curl | jq` ci-dessus fonctionne donc avec l'URL de bridge ;
- ou en tant que `ssh_ca_file` du bloc `oidc`, pour
  [les personnes dont le fournisseur se porte garant](#people-the-identity-provider-vouches-for).

Les autres clients de bridge n'écrivent pas de délégation. Si leurs certificats doivent continuer à fonctionner
ici, c'est à cela que sert `ssh_accept_ungranted`. Mais il laisse entrer **tous** les
certificats sans délégation de toutes les autorités de confiance : préférez donner une délégation à ces
clients.

### Un certificat épinglé à une adresse {#a-certificate-pinned-to-an-address}

Le `ssh_source_address` de bridge écrit l'option critique `source-address`, comme
le fait `ssh-keygen -O source-address=...`. Un certificat qui en porte une est
accepté **depuis ces adresses seulement**, avec `ssh_domains` comme sans. Une
connexion depuis une adresse que le certificat autorise est acceptée, pourvu que sa délégation
nomme aussi cet hôte quand `ssh_domains` est défini. Une connexion depuis toute autre adresse est
refusée.

| fileshare | `trusted_user_ca_file` | `ssh_ca_file` d'`oidc`, et OpenPubkey |
|---|---|---|
| avant la v0.22.0 | appliquée | **refusé depuis toutes les adresses** |
| v0.22.0, sans `ssh_domains` | appliquée | **refusé depuis toutes les adresses** |
| v0.22.0, avec `ssh_domains` | **refusé depuis toutes les adresses** | **refusé depuis toutes les adresses** |
| v0.22.1, avec ou sans `ssh_domains` | appliquée | appliquée |

« Refusé depuis toutes les adresses » échoue du côté sûr : la restriction du certificat n'a
jamais été relâchée, seule la connexion était perdue. Avant la v0.22.1, le sshd de go-filesystems/sftp
refusait toute option critique sur les certificats qu'il confiait à fileshare pour
décision ; cela couvre ceux du fournisseur, ceux d'OpenPubkey et, sous `ssh_domains`,
tous les certificats. La v0.22.1 repose sur go-filesystems/sftp v0.5.1, dont le sshd
accepte `source-address` à cet endroit et l'applique sur tous les chemins de certificats,
face à l'adresse d'où vient la connexion, IPv4 ou IPv6.

Toute autre option critique (`force-command`, `verify-required`, ...) est refusée,
puisque ce serveur n'en tient pas compte.

## Les personnes dont le fournisseur d'identité se porte garant {#people-the-identity-provider-vouches-for}

Depuis la v0.11.0, un bloc `oidc` atteint aussi SFTP — non pas avec un jeton, que SSH n'a
nulle part où mettre, mais avec un **certificat** :

```hcl
oidc {
  issuer           = "https://login.example.org"
  audience         = "fileshare"
  ssh_ca_file      = "/etc/fileshare/bridge-ca.pub"   # go-authn/bridge's ssh_ca
  opkssh_client_id = "opkssh"                         # OpenPubkey logins
  opkssh_max_age   = "24h"                            # 12h, 24h, 48h, 1week
}
```

- **Un certificat signé par l'AC SSH du fournisseur** — `bridge ssh-cert` en écrit un
  après une connexion par la fédération. Son principal est la personne, son
  extension `groups@go-authn.org` ses groupes : les
  [règles `oidc:groups:`]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})
  s'appliquent donc. Ce n'est **pas** `trusted_user_ca_file`, dont les certificats portent sur
  des comptes locaux : le certificat d'une autorité locale qui revendique les groupes du fournisseur
  est lu comme le compte local qu'il nomme.
- **Un certificat OpenPubkey**, tel que l'écrit `opkssh login` : signé par la propre
  clé de l'utilisateur, avec un jeton d'identité qui s'engage sur cette clé. Il est vérifié comme
  `opkssh verify` le vérifie — le vérificateur
  [openpubkey](https://github.com/openpubkey/openpubkey) (la signature du fournisseur
  face à ses clés publiées, l'engagement du nonce, l'identifiant client, l'âge), puis la clé du certificat
  face à celle du jeton — et le nom d'utilisateur SSH doit être celui du jeton, car il n'y a ici aucun fichier `auth_id`
  pour faire correspondre l'un à l'autre : `sftp alice@univ-example.fr@files.example.org`.
  `opkssh_client_id` devrait être un identifiant client qui lui est propre, jamais celui d'une autre
  application : le jeton d'identité voyage vers chaque serveur auquel la personne se connecte.

Pour voir les groupes que porte un certificat :

```sh
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub     # the Extensions section
```

et le serveur indique, à chaque connexion fédérée,
`sftp: alice@univ-a.fr, vouched for by the provider, in groups [...]`.

{{< callout type="error" >}}
**Garanti ne veut pas dire admis**

Quelqu'un dont le fournisseur se porte garant reste un inconnu ici, sauf si une règle
le nomme, si `trust_all` dit que le fournisseur est l'annuaire, ou si
`local_names = true` dit que les noms du fournisseur sont ceux de ce serveur et qu'un compte
local porte le sien — le même test que passe un jeton en WebDAV. Un certificat du fournisseur **sans principal**,
valable pour *n'importe qui* selon la définition même du format, est refusé.
{{< /callout >}}

`-tags noopenpubkey` laisse de côté le vérificateur OpenPubkey (environ 2 Mo) ; une
configuration avec `opkssh_client_id` dans une telle compilation est refusée.

## Reprendre un certificat {#taking-a-certificate-back}

{{< callout type="error" >}}
**Un certificat est vérifié à la connexion**

Révoquer la personne chez le fournisseur ne révoque pas, à soi seul, un
certificat déjà émis : il ouvre SFTP jusqu'à son expiration, et une session SFTP
déjà ouverte le reste jusque-là — ces personnes ne sont pas dans l'annuaire,
aucun [rechargement]({{< relref "/administration/reload.md" >}}) ne les concerne donc. La fenêtre est la
durée de vie du certificat : le `ssh_ca { validity }` de bridge (12h par défaut, au
plus 168h, jamais au-delà de la fin de la session de l'IdP) et `opkssh_max_age` ici. Gardez-la
aussi courte que le permet la reconnexion des clients. Depuis la v0.17.0, une session opkssh
déjà ouverte se termine elle aussi à `opkssh_max_age`, quoi que dise le
certificat (que la personne signe elle-même).
{{< /callout >}}

Deux mécanismes ferment cette fenêtre :

- la **[KRL]({{< relref "/security/revocation-lists.md" >}})** du fournisseur (`ssh_krl_url`) :
  un certificat révoqué est refusé à la connexion, et chaque opération d'une session qu'il a
  ouverte interroge à nouveau la liste, fichiers ouverts compris ;
- les **[signaux partagés]({{< relref "/security/shared-signals.md" >}})** (`ssf`) : tout ce
  que le fournisseur a émis pour la personne avant un `session-revoked` CAEP est refusé —
  le seul moyen d'atteindre un certificat OpenPubkey, qui ne figure dans aucune liste.

Les deux refusent en cas de doute.

## Pas de mot de passe {#no-password}

Un client qui en demande un fait précisément ce que les clés existent pour éviter.

## Taille des transferts et fichiers ouverts {#transfer-size-and-open-files}

fileshare répond à `limits@openssh.com` d'OpenSSH (depuis la v0.25.0) : le
`sftp` d'OpenSSH lit et écrit jusqu'à 255 Kio par requête au lieu de 32 Kio,
environ +25 % mesuré avec OpenSSH 10.3. Une session peut tenir au plus 1024
fichiers et répertoires ouverts à la fois ; sur un partage de répertoire,
chacun est un descripteur de l'hôte, et la limite empêche un utilisateur de
tous les prendre.
