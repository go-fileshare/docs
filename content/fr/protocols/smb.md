---
title: "SMB — NTLMv2"
linkTitle: "SMB : NTLMv2"
weight: 10
description: "SMB avec NTLMv2 : ce qu'un annuaire doit détenir, le port privilégié sous isolation, et ce qu'un rechargement de l'annuaire fait aux connexions ouvertes."
tags: [protocoles, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

Le mot de passe **ne traverse jamais le réseau** : un client envoie une preuve calculée à partir
de celui-ci. Le partage dit à un lecteur qu'il n'est que lecteur — dans le masque d'accès, avant qu'il
n'essaie, plutôt qu'en faisant échouer une écriture plus tard.

## Ce qu'un annuaire doit détenir {#what-a-directory-must-hold}

NTLMv2 a besoin du mot de passe lui-même ou de son MD4 (`sambaNTPassword`, une colonne
`nt_hash`). **Rien d'autre ne convient.**

Un annuaire qui ne fait que *vérifier* les mots de passe — une liaison LDAP, une colonne bcrypt —
ne peut pas répondre à SMB, aussi bonne que soit la vérification, car le serveur doit calculer
la même preuve que le client. C'est une propriété du protocole, pas une
limite de ce programme.

[`check`]({{< relref "/configuration/check.md" >}}) affiche un `-` dans la colonne SMB pour une telle
personne, au moment de la configuration, plutôt que de la laisser le découvrir lors d'un
montage.

## Un port privilégié sans serveur privilégié {#a-privileged-port-without-a-privileged-server}

Sous [`--isolate`]({{< relref "/operations/isolation.md" >}}) sous Unix, le parent se lie au port 445
et passe l'écouteur à l'enfant : le processus qui parle réellement SMB
n'a donc jamais besoin du privilège. Windows n'a pas d'`ExtraFiles` : là, l'enfant
se lie lui-même à l'adresse ; l'isolation est la même, la moitié « port privilégié »
ne l'est pas.

## Vérifié face à un client que ce projet n'a pas écrit {#verified-against-a-client-this-project-did-not-write}

[go-smb2](https://github.com/cloudsoda/go-smb2) mène vingt lectures concurrentes
en SMB pendant que vingt autres passent par WebDAV, sous `-race`, en CI. macOS
monte un partage en SMB pendant que `curl` écrit dans la même image en WebDAV, et
relit ce que WebDAV a écrit.

## Quand les personnes changent {#when-the-people-change}

SMB fixe qui peut se connecter à un partage **au moment où le partage est ajouté**, et le vérifie
une fois par connexion d'arborescence (tree connect). Une nouvelle personne trouvée par un [rechargement de l'annuaire]({{< relref "/administration/reload.md" >}})
est donc ajoutée sur place, serveur SMB en cours d'exécution compris, sans toucher aucune connexion ;
mais tout ce qui est retiré — une personne, un identifiant, les listes développées d'un partage —
constitue une nouvelle génération, et les connexions SMB ouvertes sont fermées : une
révocation atteint ainsi les sessions déjà ouvertes. Un partage dont le groupe `allow` s'est
vidé **n'est pas du tout proposé en SMB** : son `AllowUsers` vide se lirait comme
« tout le monde ».
