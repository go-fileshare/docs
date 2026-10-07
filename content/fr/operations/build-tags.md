---
title: "Ne compiler que ce que l'on veut"
weight: 10
description: "Les étiquettes de compilation qui laissent hors du binaire des protocoles, des annuaires d'utilisateurs, l'API d'administration et l'approvisionneur, et ce que chacune fait gagner."
tags: [exploitation, compilation]
---

Chaque protocole est derrière une étiquette de compilation (build tag), et une étiquette le laisse
**entièrement** de côté : ni écouteur, ni analyseur, ni dépendance, ni code.

```sh
go install -tags nonfs,nowebdav,nosftp,nos3 github.com/go-fileshare/fileshare@latest   # SMB only
go build   -tags nosmb,nonfs,nosftp,nos3 .                                            # WebDAV only
```

La provenance des **personnes** a ses propres étiquettes : `nosql` laisse de côté
les trois pilotes de base de données, `noldap` le client LDAP.

| compilation | taille |
|---|---|
| tout | 31,2 Mo |
| `-tags noldap` | 30,9 Mo |
| `-tags nosftp` | 30,6 Mo |
| `-tags noopenpubkey` (pas de connexion opkssh en SFTP) | environ 2 Mo de moins |
| `-tags nos3` | 31,1 Mo |
| `-tags nonfs,nowebdav,nosftp,nos3` (SMB seul) | 28,1 Mo |
| `-tags nopartitioned` (ni apfs, ni btrfs, ni xfs, ni zfs) | 29,2 Mo |
| `-tags nosql` | 20,2 Mo |
| `-tags nosql,noldap` | 19,8 Mo |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp,nos3` | 16,2 Mo |

Ce sont les chiffres que donne le README de fileshare, mesurés avant l'arrivée de l'API
d'administration ; le tableau ci-dessous est la mesure suivante.

## L'API d'administration, et `nogrpc` {#the-admin-api-and-nogrpc}

L'[API d'administration]({{< relref "/administration/_index.md" >}}) a fait entrer gRPC et protobuf, et
ils sont derrière `nogrpc` (linux/amd64, mesuré le 2026-09-29, quand les lignes ci-dessus
étaient passées à 34,1 Mo pour tout) :

| compilation | taille |
|---|---|
| tout, avec l'API d'administration | 46,1 Mo |
| `-tags nogrpc` | 34,4 Mo |
| `-tags nosql,noldap,nogrpc` | 22,5 Mo |

gRPC coûte **11,7 Mo** — la [mesure des greffons](#why-not-subprocess-plugins)
ci-dessous, refaite, maintenant qu'il est là pour une raison qui lui est propre. Un site qui
gère ses partages dans des fichiers n'a pas besoin de le transporter.

{{< callout type="warning" >}}
**Un bloc `admin` dans une compilation `nogrpc` est refusé**

Plutôt que servi sans API : une configuration qui demande l'API et
démarre sans elle ne fait pas ce qu'elle dit.
{{< /callout >}}

`nogrpc` laisse de côté l'API d'administration et, avec elle,
l'[approvisionneur]({{< relref "/administration/volumes.md" >}}). [Santé et métriques]({{< relref "/administration/health.md" >}})
sont du simple HTTP et restent.

`-tags noprovisioner` ne laisse de côté que l'approvisionneur (provisioner), le rôle privilégié
`fileshare provisioner`, et avec eux zfs, btrfs et projquota de go-fsctl :
un serveur qui ne sert jamais de volumes n'a pas à transporter le code qui les crée.
`fileshare provisioner` dans une telle compilation le dit, plutôt que « commande inconnue ».

`nosql` est de loin le levier le plus puissant : PostgreSQL, MySQL et SQLite
pèsent ensemble **11,7 Mo**, plus que tous les protocoles de ce programme
réunis. Un site dont les utilisateurs sont dans le fichier le voudra. Les pilotes sont importés
par la commande et non par la bibliothèque qui les utilise : c'est ce qui fait du
choix une étiquette de compilation plutôt qu'un fork.

{{< callout type="info" >}}
**Un protocole avec lequel ce binaire n'a pas été compilé**

Une configuration qui en nomme un se le voit dire *tel quel*, plutôt que « ce protocole
n'existe pas » — la différence entre une faute de frappe et une étiquette de compilation. Il en va de même
pour `opkssh_client_id` sous `noopenpubkey` et pour un bloc `admin` sous
`nogrpc`.
{{< /callout >}}

## Pourquoi pas des greffons en sous-processus {#why-not-subprocess-plugins}

Cela a été **mesuré plutôt que débattu** : `hashicorp/go-plugin` apporte gRPC et
protobuf, qui coûtent **13,2 Mo à eux seuls** — plus que tout ce binaire,
avec ses trois protocoles et tous ses pilotes. Un hôte de greffons ferait le double
de la taille avant d'avoir chargé quoi que ce soit, et chaque binaire de greffon transporterait encore
gRPC.

Pour la surface d'attaque, une étiquette est aussi l'outil le plus solide pour tout ce que vous
n'exécutez pas : **du code qui n'a jamais été compilé ne peut pas être atteint**, en bac à sable ou non.

Ce que les étiquettes ne donnent *pas*, c'est l'isolation entre les protocoles que vous exécutez **bel et bien**,
ni un moyen d'ajouter un protocole sans recompiler. Les deux sont de vrais besoins, et tous deux
plaident pour une frontière de processus plutôt que pour un binaire plus petit — c'est
[un processus par protocole]({{< relref "/operations/isolation.md" >}}), avec ce même binaire, et non un
cadre de greffons.
