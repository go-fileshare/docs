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

## Les images HFS+ sont inscriptibles {#hfsplus-images-are-writable}

Depuis la v0.44.0 (hfsplus v0.9.0), un partage dont l'image contient du HFS+ est
ouvert en écriture, comme FAT, exFAT et ext4 : un client peut créer, modifier et
supprimer des fichiers, et les dates, modes et propriétaires qu'il fixe sont
enregistrés. Auparavant, une image HFS+ était ouverte en lecture seule. L'image
est lue en mémoire à l'ouverture du partage et chaque modification est réécrite
dans le fichier.

**Ce que ça coûte.** hfsplus v0.9.0 garde toute l'image en mémoire et la réécrit
entièrement à chaque opération (création, écriture, renommage, `Chtimes`) : la
mémoire d'un partage est la taille de son image, et l'E/S de chaque modification
aussi (go-filesystems/hfsplus#33 est la vraie correction). Un partage n'ouvre donc
une image HFS+ en écriture que jusqu'à `hfsplus_writable_max`, 256 Mio par défaut
(`"0"` jamais ; un nombre d'octets, ou un nombre suivi de K, M, G ou T) :

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"
}
```

Au-delà, ou si la taille est inconnue, le partage s'ouvre en **lecture seule**,
toujours avec la lecture par plages et les dates, et le serveur écrit une ligne
qui nomme l'image, la taille et la limite :

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes): a writable HFS+ image is held in memory and rewritten whole on every change, so mac is read-only
```

Un volume **à journal** (créé par `hdiutil create -fs "Journaled HFS+"` ou par
macOS) n'est écrit que si son journal est vide, ce que laisse un démontage
propre. Un journal qui contient des transactions, ou qu'on ne peut pas vérifier,
serait rejoué par macOS par-dessus ce que le serveur écrit : le pilote le refuse
donc, et **le partage s'ouvre en lecture seule à la place**. Le serveur démarre
quand même ; à l'ouverture du partage il écrit une ligne qui nomme l'image, le
partage et la raison :

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

Pour rendre un tel partage inscriptible, il faut d'abord faire rejouer le
journal : monter l'image sur un Mac puis l'éjecter proprement, ou lancer
`fsck_hfs` dessus. (Le pilote n'a pas de vérification du journal qui marche sans
l'image en mémoire : ce seul cas lit l'image, jusqu'à la limite, avant de se
rabattre.) Un partage avec `read_only = true`, un périphérique et une partition
n'essaient jamais l'ouverture en écriture.

**Un autre écrivain.** L'image est réécrite entière : un second écrivain serait
écrasé par la réécriture suivante. L'ouverture en écriture prend un verrou
consultatif exclusif `flock` sur l'image (Linux, macOS et les BSD) ; si un autre
processus le détient, par exemple un second fileshare sur la même image, le
partage s'ouvre en lecture seule et le dit. Le verrou est consultatif : un
montage macOS ne le prend pas, et sur les autres plateformes il n'y a pas de
verrou.

⛔ **N'écrivez pas dans une image montée**, ni avec ce serveur ni avec autre
chose : aucun des deux ne connaît les modifications de l'autre. Servez une image
que rien d'autre n'a ouverte.

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

## Ce que fait SMB d'une date qu'il ne peut pas enregistrer {#what-smb-does-with-a-time-it-cannot-store}

Une date qu'un client fixe et que le stockage ne peut pas conserver est
**ignorée** par défaut. Depuis la v0.42.0, un partage peut choisir, avec
`unstorable_times`. Ne sont pas enregistrables : une CreationTime différente de
celle annoncée, une ChangeTime envoyée seule, ou une date d'accès ou d'écriture
sur une image dont le pilote ne sait pas fixer les dates (NTFS,
APFS, ISO 9660, SquashFS ; un répertoire de l'hôte et une image FAT, exFAT, ext4 ou HFS+ les enregistrent) :

```hcl
share "photos" {
  directory        = "/data/photos"
  unstorable_times = "log"      # "ignore" (par défaut), "refuse" ou "log"
}
```

`refuse` répond au client par une erreur, et `log` écrit une ligne qui nomme le
partage et le chemin. Les cas exacts, selon le type de partage, sont dans
[Dates réelles des fichiers]({{< relref "/protocols/times.md#when-unstorable" >}}).
