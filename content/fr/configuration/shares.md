---
title: "Partages : images, périphériques, répertoires"
weight: 10
description: "Ce qu'un partage peut servir (une image disque, un périphérique ou un répertoire de l'hôte) et comment son système de fichiers et sa partition sont choisis."
tags: [configuration, partages]
---

Un partage sert l'une de trois choses : une **image disque**, un nœud de **périphérique**, ou un
**répertoire de l'hôte**. Une image et un périphérique portent un système de fichiers que ce
programme lit lui-même ; un répertoire est celui de l'hôte.

## Une image, et le système de fichiers qu'elle contient {#an-image-and-the-filesystem-inside-it}

Le système de fichiers que contient une image est **déduit plutôt que déclaré**.
[`go-filesystems/detect`](https://github.com/go-filesystems/detect) lit le
nombre magique et renvoie le pilote qui lui correspond : **fat32, exfat, ext4, ntfs, ufs,
iso9660, squashfs ou hfsplus** — tous les pilotes de la même forme,
`OpenReader(io.ReaderAt, int64)`, un système de fichiers à l'offset zéro.

## Les quatre qu'on ne peut pas flairer {#the-four-that-cannot-be-sniffed}

**apfs, btrfs, xfs et zfs** ouvrent chacun une image de *disque* et choisissent une **partition** :
il n'y a donc aucun nombre magique à trouver à l'offset zéro. Un partage dit lequel :

```hcl
share "photos" {
  image      = "/srv/disk.img"
  filesystem = "xfs"
  partition  = 2       # or leave it out: the driver takes the first data partition
}
```

## Une partition, pour n'importe quel système de fichiers {#a-partition-for-any-filesystem}

Une image disque qui contient du FAT32, de l'ext4 ou de l'exFAT a une table de partitions devant elle
aussi souvent qu'une image qui contient du XFS — et la détection lit l'offset zéro, où une
image partitionnée a la *table*. La partition est donc choisie d'abord, et le
pilote reçoit une vue de celle-ci :

```hcl
share "photos" {
  image           = "/srv/disk.img"
  partition_label = "photos"          # what lsblk calls PARTLABEL
}
```

Trois façons d'en nommer une, et **une seule peut être donnée** — deux qui se contrediraient
serviraient celle que le code essaierait en premier :

| | |
|---|---|
| `partition = 2` | en comptant à partir de 1, comme les outils de partitionnement les affichent |
| `partition_label = "photos"` | le nom de partition GPT |
| `partition_uuid = "…"` | le GUID unique GPT — le `PARTUUID` de Linux |

{{< callout type="error" >}}
**Un index bouge**

Un disque repartitionné, un outil qui écrit les entrées dans un autre ordre, une image
restaurée avec une partition de moins — et `partition = 2` désigne autre
chose, **en silence**, parce qu'on y trouve toujours un système de fichiers. Une étiquette ou un
UUID désigne la partition elle-même : c'est pourquoi les fstab ont cessé d'utiliser des index
il y a des années. L'index est là pour les images MBR, qui n'ont ni l'un ni l'autre.
{{< /callout >}}

Un partage qui a choisi une partition est **en lecture seule**, et le dit au démarrage : les
offsets du pilote sont ceux de la partition alors que le fichier sous-jacent est le disque
entier, si bien qu'une écriture atterrirait à cet offset depuis le début de l'*image* — sur
la table de partitions, bien souvent.

## Nommer un système de fichiers désactive la détection {#naming-a-filesystem-turns-detection-off}

{{< callout type="error" >}}
**C'est le but, et c'est le risque**

L'image est ouverte comme *ce* système de fichiers, ou refusée. Une image FAT32 à qui l'on dit qu'elle est XFS
ne devient pas un partage XFS ; elle ne démarre pas, et dit comme quoi on lui a demandé
de l'ouvrir.
{{< /callout >}}

`filesystem` est accepté aussi pour ceux qu'on peut flairer, et les deux sont alors
comparés : un partage qui dit `ext4` sur une image FAT32 est refusé avec *the
share says ext4 and the image holds fat32*. C'est ainsi qu'un site refuse une
détection erronée plutôt que de la découvrir plus tard.
[`check`]({{< relref "/configuration/check.md" >}}) marque d'un astérisque un pilote nommé, parce que cette ligne n'a pas été
reconnue — elle a été affirmée.

## Un périphérique, pas seulement une image {#a-device-not-only-an-image}

```hcl
share "photos" {
  image = "/dev/sda"        # a device node, not a file
}
```

**Aucun privilège n'est en jeu.** Rien ici ne monte quoi que ce soit — les pilotes
[`go-filesystems`](https://github.com/go-filesystems) lisent ext4, xfs et
les autres en espace utilisateur — il n'y a donc pas de `mount(2)` et par conséquent pas de
`CAP_SYS_ADMIN`. Ouvrir un périphérique est un `open(2)` ordinaire, régi par les
permissions du nœud :

| | propriétaire | mode | suffisant |
|---|---|---|---|
| Linux `/dev/sda` | `root:disk` | 0660 | appartenance à `disk` |
| macOS `/dev/disk0` | `root:operator` | 0640 | appartenance à `operator`, plus l'accès complet au disque (Full Disk Access) |

Un refus dit quel groupe, plutôt que `permission denied` — la réponse n'est
jamais `sudo`.

⛔ **Un partage de périphérique est en lecture seule**, pour la même raison qu'un partage qui a choisi une
partition : écrire sur un disque en service n'est pas une décision à prendre à votre place.

{{< callout type="error" >}}
**Le périphérique est ouvert en exclusivité, et c'est une question de justesse, pas de prudence**

Si le noyau a un système de fichiers monté depuis ce périphérique, le cache de pages du système de fichiers
détient des métadonnées plus récentes que le périphérique — si bien qu'un lecteur brut voit un
bloc de répertoire d'avant une mise à jour à côté d'un bloc d'inode d'après celle-ci,
un état qui n'a jamais existé sur le disque à aucun moment. `O_EXCL` sur un périphérique
bloc est la primitive du noyau lui-même pour « personne d'autre, montage compris » ;
c'est ce qu'utilisent `mkfs` et `fsck`. Un périphérique en cours d'utilisation est refusé, nommément, avec
la raison.
{{< /callout >}}

Deux choses ont été mesurées plutôt que supposées :

- `Stat().Size()` vaut **0** pour un périphérique : la longueur est donc demandée au périphérique
  lui-même. Se positionner à la fin répond sous Linux et renvoie **0 sans erreur**
  sous macOS : c'est pourquoi Darwin utilise `DKIOCGETBLOCKCOUNT` à la place.
- Un nœud **brut** (`/dev/rdiskN`, ou `O_DIRECT` sous Linux) refuse une lecture
  non alignée, et lire un champ de deux octets à l'offset 11, c'est précisément ce *qu'est* l'analyse d'un BPB FAT.
  Les lectures sur un périphérique sont arrondies à des blocs entiers ; sans cela,
  `/dev/rdisk4` signalait `unknown filesystem`.

## Un répertoire, pas seulement une image {#a-directory-not-only-an-image}

Depuis la v0.8.0, un partage peut servir un répertoire de l'hôte — le volume d'un conteneur,
par exemple — plutôt qu'une image :

```hcl
share "photos" {
  directory = "/data/photos"
  allow     = ["@family"]
}
```

C'est [go-filesystems/osfs](https://github.com/go-filesystems/osfs), qui
atteint l'arborescence par un **`os.Root`** : un `..` qui en sort, un chemin
absolu, et un lien symbolique qui mène dehors sont refusés par la racine adossée au noyau,
pas par un test sur des chaînes. Un client peut *créer* un lien vers `/etc` ; rien ne le
suivra. Une FIFO plantée dans l'arborescence est refusée plutôt que laissée bloquer le
serveur.

{{< callout type="error" >}}
**Un partage ne peut pas en contenir un autre (depuis la v0.17.0)**

Un partage de répertoire dont l'arborescence contient l'image ou le répertoire d'un autre partage est
**refusé au démarrage**, ainsi que par l'API d'administration : quiconque peut utiliser le partage
extérieur lirait et écrirait l'intérieur sans y être autorisé —
NFS anonyme compris. De même pour deux partages sur la même source, sauf deux
partitions différentes d'une même image disque.
{{< /callout >}}

{{< callout type="info" >}}
**Pas derrière le verrou d'image**

Un pilote d'image possède un seul fichier et ne promet rien sur deux appels simultanés,
c'est pourquoi [chaque image est enveloppée dans un verrou]({{< relref "/operations/locking.md" >}}).
Une arborescence de l'hôte appartient au noyau, qui sérialise ce qui doit l'être, appel
par appel, et pas davantage ; un seul mutex devant chaque fichier de chaque client serait
la chose la plus lente que puisse faire un serveur de fichiers.
{{< /callout >}}

⛔ Un partage a une `image` **ou** un `directory`, jamais les deux, et jamais aucun des deux.
`filesystem` et `partition` concernent une image : un partage de répertoire qui en nomme un est
refusé, parce que le dire signifie que le partage devait être une image.
`read_only = true` fonctionne sur un répertoire comme sur une image ; un partage qui le dit
et liste aussi des `writers` est refusé, parce que `read_only` l'emporterait.

La capacité annoncée à un client est celle du système de fichiers sur lequel se trouve l'arborescence ; une
plateforme qui ne sait pas la dire la laisse à zéro, ce que les protocoles lisent comme
« inconnue » plutôt que « pleine ».
