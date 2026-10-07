---
title: "Utilisateurs, groupes et annuaires"
weight: 20
description: "D'où viennent les personnes (blocs user et group, SQL, LDAP, un fournisseur d'identité) et ce que chaque source leur permet d'utiliser."
tags: [configuration, identité, oidc, ldap, sql]
---

Un bloc `user` est tout l'annuaire d'un foyer. Un site dont les personnes sont
déjà dans une base de données ou dans LDAP ne devrait pas les recopier dans un second endroit qui
se périme : un bloc `users` les lit donc là où elles sont.

```hcl
users "sql" {
  driver   = "postgres"                 # or sqlite, or mysql
  dsn_file = "/etc/fileshare/dsn"       # a DSN holds a password: it lives in a file
  users    = "select login, null, nt_hash, ssh_keys from staff"
  groups   = "select team, member from team_members"
}

users "ldap" {
  url                = "ldaps://ldap.example.org"
  base_dn            = "ou=people,dc=example,dc=org"
  bind_dn            = "cn=reader,dc=example,dc=org"
  bind_password_file = "/etc/fileshare/bind.pw"
}
```

Les **requêtes vous appartiennent**, parce que les personnes d'un site ont déjà la forme de ce site ;
un schéma inventé ici obligerait à les recopier dans un second. Les colonnes de la
requête des personnes sont lues **par position**, pas par nom : un nom, puis
tout ou partie d'un mot de passe, d'une empreinte NT, de clés SSH (une par ligne) et d'un secret TOTP, dans
cet ordre. Un NULL est un identifiant que cette personne n'a pas : c'est pourquoi
l'exemple sélectionne `null` pour le mot de passe. Une colonne de mot de passe est prise comme le
mot de passe lui-même : il n'y a ici aucun réglage pour un mot de passe haché.

Côté LDAP, c'est ce qu'un annuaire compatible Samba publie déjà qui est lu —
`sambaNTPassword`, `sshPublicKey`, `memberUid`. L'emplacement des personnes et des groupes,
et les attributs qui les nomment, sont configurables : `user_filter`
(par défaut `(objectClass=posixAccount)`), `user_attribute` (`uid`),
`group_base_dn` (`base_dn`), `group_filter` (`(objectClass=posixGroup)`),
`group_attribute` (`cn`), `group_member_attribute` (`memberUid`), et
`totp_attribute`, qui n'a pas de valeur par défaut. `sambaNTPassword` et `sshPublicKey`
sont lus sous ces noms.

{{< callout type="error" >}}
**Pas de liaison en clair vers une autre machine (depuis la v0.19.0)**

Un bloc `users "ldap"` refuse `ldap://` vers une autre machine sauf si
`start_tls = true` : chaque mot de passe que vérifie une liaison traverserait le réseau
en clair. `ldaps://` est accepté, de même que `ldap://` vers une adresse de boucle
locale et `ldapi://`, où personne ne se trouve sur le chemin. Il n'y a aucun interrupteur pour
désactiver cela (go-authn/directory v0.10.0).
{{< /callout >}}

## L'ordre, et la seule exception {#order-and-the-one-exception}

Les sources sont interrogées **dans l'ordre où elles sont écrites**, et la première qui
connaît un nom en est propriétaire. Les blocs `user` et `group` viennent en premier : un compte
de service inscrit localement n'est donc pas supplanté par quelqu'un du même nom
dans LDAP.

**Les groupes sont l'exception** : les membres d'un groupe sont l'**union** de toutes les
sources, parce qu'une équipe peut avoir des personnes dans un fichier et dans une base de données.

```hcl
share "photos" {
  allow   = ["@engineers", "alice"]
  writers = ["@owners"]
}
```

{{< callout type="info" >}}
**Développés au démarrage, et de nouveau à chaque rechargement**

Depuis la v0.10.0, les blocs `users` — SQL, LDAP — sont relus à chaque
`reload = "5m"`, sur `SIGHUP`, et sur le `ReloadDirectory` de l'API d'administration ; une
appartenance qui change dans LDAP n'attend plus un redémarrage. Ce qu'un rechargement
applique sur place et ce qu'il redémarre, et pourquoi un groupe qui se vide
n'ouvre **pas** son partage à tout le monde, se trouvent dans
[relire l'annuaire]({{< relref "/administration/reload.md" >}}). L'annuaire
n'est toujours pas interrogé à chaque connexion : c'est une autre conception, et ce
n'est pas celle-ci.
{{< /callout >}}

## Ce qu'une source peut prouver, et ce dont chaque protocole a besoin {#what-a-source-can-prove-and-what-each-protocol-needs}

C'est la partie qu'un site découvre sinon lors d'un montage : [`check`]({{< relref "/configuration/check.md" >}})
la dit donc en premier :

| la source a | SMB | S3 | WebDAV | SFTP |
|---|---|---|---|---|
| un mot de passe (fichier, ou colonne en clair) | oui | oui | oui | — |
| une empreinte NT (`sambaNTPassword`, `nt_hash`) | oui | **non** | — | — |
| seulement une liaison (LDAP) | **non** | **non** | oui | — |
| des clés publiques, ou une AC de confiance | — | — | — | oui |

{{< callout type="error" >}}
**NTLMv2 a besoin du mot de passe ou de son MD4, et rien d'autre ne convient**

Un client n'envoie jamais de mot de passe à un serveur SMB — il envoie une preuve
calculée à partir de celui-ci — si bien qu'un annuaire qui ne fait que *vérifier* les mots de passe ne peut pas
répondre à SMB, aussi bonne que soit la vérification. C'est une propriété du protocole,
pas une limite de ce programme, et aucune configuration n'y
change rien. WebDAV demande seulement « est-ce le bon mot de passe », ce à quoi une liaison répond.
{{< /callout >}}

Une personne qu'un annuaire nomme sans rien prouver pour elle est légitime — un listage
avec les secrets ailleurs — et `check` le dit en une ligne plutôt que de
la laisser le découvrir.

## Les personnes que nomme le fournisseur d'identité, pas ce fichier {#people-the-identity-provider-names-not-this-file}

Une fédération — RENATER par l'intermédiaire de
[go-authn/bridge](https://github.com/go-authn/bridge), par exemple — sait qui fait partie d'un
projet, et un partage peut le lui demander à **elle** plutôt que recopier la liste. Avec un
[bloc `oidc`]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}) :

```hcl
share "photos" {
  image   = "/srv/photos.img"
  allow   = ["oidc:groups:urn:mace:univ-example.fr:photos", "oidc:user:bob@univ-example.fr", "alice"]
  writers = ["oidc:groups:urn:mace:univ-example.fr:photos"]
}
```

`oidc:groups:<value>` désigne quelqu'un dont la revendication de groupes du jeton contient la valeur ;
`oidc:user:<name>` désigne quelqu'un que le jeton nomme. La graphie est celle d'opkssh, pour
qu'un seul vocabulaire dise qui atteint un shell et qui atteint un partage. Quelqu'un
que nomme une règle est connu de ce serveur pour autant que le fournisseur le garantisse — les partages
ouverts compris — et quelqu'un qu'aucune règle ne nomme reste un inconnu. Les mêmes règles
s'appliquent en [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}),
où les groupes voyagent dans le certificat plutôt que dans le jeton.

{{< callout type="error" >}}
**Une règle ne concerne que les personnes du fournisseur**

`oidc:user:bob` est le bob dont le fournisseur se porte garant ; un compte local nommé
bob, avec un mot de passe, est quelqu'un d'autre et ne correspond à aucune règle. Une règle sans
bloc `oidc`, une règle mal formée, ou une règle qui permettrait d'écrire sans autoriser
la connexion est **refusée au démarrage**.
{{< /callout >}}

{{< callout type="error" >}}
**L'`alice` du fournisseur n'est pas l'`alice` locale, sauf si vous le dites (depuis la v0.20.0)**

Un nom simple dans `allow` ou `writers` est un compte local ; un jeton ou un
certificat du fournisseur n'est atteint que par les règles `oidc:` d'un partage. Un site
dont les noms du fournisseur SONT ses noms locaux écrit `local_names = true` dans
le bloc `oidc` — et devrait définir `domains` avec, ou prendre le nom dans
une revendication que le fournisseur contrôle : un fournisseur où chacun choisit son propre
`preferred_username` donnerait sinon à quiconque s'inscrit les partages d'un compte
local, écriture comprise. C'était le comportement par défaut avant la v0.20.0
(audit de sécurité F4). Un nom fédéré refusé faute de `local_names`
est journalisé avec la ligne à ajouter, et `fileshare check` dit quelle règle est
en vigueur.

Les certificats clients NFS (`identity = "certificate"`) sont l'exception :
leur AC est celle que `client_ca_file` épingle précisément pour cela, et les noms simples
sont le seul moyen pour un partage NFS de nommer quelqu'un : leurs noms sont donc
des noms locaux, quoi que dise `local_names`.
{{< /callout >}}

### Quels établissements {#which-institutions}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
  domains  = ["univ-a.fr", "univ-b.fr"]   # nobody else from the federation gets in
}

share "projet-x" {
  image   = "/srv/projet-x.img"
  allow   = ["oidc:groups:urn:mace:univ-a.fr:projet-x", "oidc:domain:univ-b.fr"]
  writers = ["oidc:groups:urn:mace:univ-a.fr:projet-x"]
}
```

`domains` est vérifié à l'authentification, en SFTP comme en WebDAV, et pour les
certificats clients NFS : un nom doit
être de la forme `<something>@<one of them>`, comparé en entier (`evilunivb.fr` n'est pas
`univb.fr`). `oidc:domain:` est le même test pour un seul partage. On peut faire confiance au domaine
autant qu'au fournisseur : go-authn/bridge écarte un eppn ou un subject-id
dont les métadonnées de fédération de l'IdP ne lui accordent pas la portée (scope).

**Les groupes** sont ce que l'IdP de l'établissement publie, transformé en revendication
`groups` par le `claims { groups = [...] }` de go-authn/bridge : `eduPersonEntitlement`
par défaut (les groupes d'un laboratoire ou d'une VO), ou `eduPersonScopedAffiliation`
(`staff@univ-a.fr`, `student@univ-b.fr`).
