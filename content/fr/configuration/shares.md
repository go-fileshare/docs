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

## Les images HFS+ sont en lecture seule tant qu'un partage n'y consent pas {#hfsplus-images-are-writable}

Depuis la v0.44.0 (hfsplus v0.9.0), un partage dont l'image contient du HFS+ peut
être ouvert en écriture, comme FAT, exFAT et ext4 : un client peut créer,
modifier et supprimer des fichiers, et les dates, modes et propriétaires qu'il
fixe sont enregistrés. Auparavant, une image HFS+ était ouverte en lecture seule.

**Désactivé par défaut depuis la v0.45.1, et toujours désactivé.** hfsplus v0.9.0
ne résistait pas aux plantages (il gardait toute l'image en mémoire, la
réécrivait entièrement à chaque opération et ne marquait jamais le volume comme
utilisé) : la v0.45.1 a donc désactivé l'écriture, que la v0.44.0 et la v0.45.0
activaient par défaut (jusqu'à 256 Mio). **Depuis la v0.46.0 (hfsplus v0.10.0)**,
un volume HFS+ inscriptible est écrit **en place** :

- **Aucun coût en mémoire.** L'image n'est pas copiée en mémoire (un cache de
  lecture de 8 Mio) ; seuls les secteurs modifiés par une validation sont écrits.
  Le plafond de 64 Gio a disparu.
- **Un plantage est visible.** Chaque validation marque le volume comme utilisé :
  le bit « démonté » et le bit « journalisé » sont effacés dès la première
  écriture de l'en-tête et rétablis à la fin, si bien que `fsck_hfs -q` voit un
  plantage, volumes à journal compris.
- **Une écriture échouée arrête le volume.** Après un vidage échoué, le volume
  refuse les écritures (`ErrVolumeFailed`) jusqu'à la réouverture du partage.

Cela reste **sur consentement**, à cause de ce que hfsplus n'a pas terminé : une
petite écriture durable coûte 3 fsync par validation et est environ 3 fois plus
lente que sur macOS (go-filesystems/hfsplus#35), les tests de plantage ne sont
pas encore durcis (go-filesystems/hfsplus#37), et après un plantage un volume à
journal se monte sans son journal jusqu'à `diskutil enableJournal`. Le défaut
pourra revenir quand cela sera corrigé. Sauvegardez d'abord l'image. Un partage y
consent avec `hfsplus_writable_max`, `"0"` par défaut (jamais ; un nombre
d'octets, ou un nombre suivi de K, M, G ou T) :

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"   # consentir
}
```

Depuis la v0.46.0, la taille ne borne plus la mémoire. Elle reste une taille pour
qu'une configuration écrite pour la v0.44.0 à la v0.45.1 garde son sens (un
partage qui disait `"256M"` ne se met pas à écrire une image de 2 Tio) ; une
grande valeur comme `"1T"` est le simple consentement.

Avec le défaut, au-delà de la taille, ou si la taille est inconnue, le partage
s'ouvre en **lecture seule**, toujours avec la lecture par plages et les dates,
et le serveur écrit une ligne qui nomme l'image et la raison, par exemple sans
réglage :

```text
/srv/mac.img is an HFS+ volume and hfsplus_writable_max is 0, the default: HFS+ images are written only when a share sets a size, because a durable small write is about 3x slower than on macOS (go-filesystems/hfsplus#35), the crash tests are not finished (go-filesystems/hfsplus#37), and after a crash a journaled volume mounts without its journal until diskutil enableJournal, so mac is read-only
```

et au-delà de la taille :

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes), so mac is read-only
```

**Un rechargement suit le réglage (depuis la v0.46.0).** Quand un rechargement
change le fait qu'une image HFS+ soit écrite (réglage retiré, ajouté, ou déplacé
de part et d'autre de la taille de l'image), l'image est rouverte : retirer le
consentement rend le partage aussitôt en lecture seule, l'ajouter le rend
inscriptible. Avant la v0.46.0, le pilote ouvert était conservé : un partage
restait inscriptible après le retrait de son consentement, jusqu'au redémarrage
du serveur. Passer d'une taille inscriptible à une autre ne change rien.

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
`fsck_hfs` dessus. Pendant l'écriture d'un volume à journal, son bit « journalisé »
est effacé ; après un plantage il le reste, et le volume se monte donc sans
journal jusqu'à `diskutil enableJournal`. Un partage avec `read_only = true`, un
périphérique et une partition n'essaient jamais l'ouverture en écriture.

Le volume est écrit par le descripteur de fichier propre du partage (hfsplus
v0.11.0) : le fichier déjà confiné et verrouillé, jamais le nom de l'image rouvert.
Une taille qui ne correspond pas au volume ouvre le partage en lecture seule,
avec une ligne.

**Un autre écrivain.** Un second écrivain sur la même image la corromprait.
L'ouverture en écriture prend un verrou consultatif exclusif `flock` sur l'image
(Linux, macOS et les BSD) ; si un autre processus le détient, par exemple un
second fileshare sur la même image, le partage s'ouvre en lecture seule et le
dit. Le verrou est consultatif : un montage macOS ne le prend pas, et sur les
autres plateformes il n'y a pas de verrou.

⛔ **N'écrivez pas dans une image montée**, ni avec ce serveur ni avec autre
chose : aucun des deux ne connaît les modifications de l'autre. Servez une image
que rien d'autre n'a ouverte.

## Une image inscriptible que l'OS a montée est servie en lecture seule {#an-image-the-os-has-mounted}

Depuis v0.45.0, avant d'ouvrir en écriture **n'importe quel** partage d'image
inscriptible (FAT32, exFAT, ext4, UFS, btrfs, XFS, ZFS, HFS+, quel que soit le
pilote), le serveur demande à l'OS s'il a cette image attachée ou montée. Si
oui, le partage s'ouvre en **lecture seule**, est marqué comme tel, et le
serveur écrit une ligne nommant l'image, le périphérique et le montage de l'OS,
et le partage :

```text
/srv/disk.img is in use by the OS (/dev/loop3, mounted at /mnt/disk), and a second writer would corrupt it, so disk is read-only
```

Le `flock` consultatif décrit plus haut ne peut pas le faire : l'OS ne prend
aucun verrou quand il monte une image. L'image est reconnue par périphérique et
inode, jamais en comparant des chaînes de chemin ; un lien symbolique ou un lien
physique vers le fichier monté est donc aussi détecté.

| OS | Comment on le demande |
| --- | --- |
| Linux | `/sys/block/loop*/loop/backing_file` de chaque périphérique loop (ce que créent `mount -o loop`, `losetup` et udisks) ; le point de montage vient de `/proc/self/mountinfo`. Un périphérique loop attaché compte même si rien n'y est monté. |
| macOS | `hdiutil info -plist`, le `image-path` de chaque image attachée, un appel par chargement, borné à 15 s. Une image attachée avec `-nomount` compte. |
| FreeBSD, autres BSD, Windows | **Détection impossible.** Le partage est inscriptible, comme avant. |

**Quand la vérification elle-même échoue** (pas de `/sys/block` dans un
conteneur, `hdiutil` absent ou dépassant le délai, sortie qui n'est pas une liste
de propriétés), personne ne sait si l'image est montée. `image_mount_unknown`
tranche :

```hcl
share "disk" {
  image               = "/srv/disk.img"
  image_mount_unknown = "read_only"   # la valeur par défaut ; "write" la sert quand même inscriptible
}
```

La valeur par défaut est `read_only` : un partage en lecture seule est un état
visible, avec une ligne de journal qui en nomme la cause, et qu'on peut corriger ;
un partage inscriptible sous un montage corrompt les données des deux côtés, et
personne ne le voit avant plus tard. `write` convient à un hôte où la
vérification ne peut pas fonctionner (un conteneur sans sysfs) et où personne ne
monte d'images. Une image que l'OS a, à coup sûr, montée reste en lecture seule
quoi que dise ce réglage. Un chemin de support que la vérification ne peut pas
lire (permission refusée, boucle de liens symboliques) est « non jugé », pas
« une autre image » : il va au même réglage. Seul un chemin qui n'existe pas est
ignoré. Sous Linux, un périphérique loop dont le fichier de support a été
supprimé est reconnu par l'enregistrement du noyau (`LOOP_GET_STATUS64`) si le
processus peut ouvrir `/dev/loopN`, sinon il n'est pas reconnu.

**Ce que la vérification ne voit pas**

- **Un montage fait après l'ouverture de l'image par le partage.** La
  vérification a lieu à l'ouverture d'une image : au démarrage, pour un partage
  créé par l'API d'administration, et quand un rechargement reconstruit les
  partages (changement de personnes ou de listes) et trouve l'image montée entre
  temps ; ce partage reçoit alors sa propre ouverture en lecture seule. Un
  rechargement qui ne change rien ne regarde pas à nouveau, et rien ne surveille
  un `hdiutil attach` ou `mount -o loop` ultérieur. Arrêtez le partage, ou
  redémarrez le serveur, avant de monter une image qu'il sert en écriture.
- **Les montages qui ne passent ni par un périphérique loop ni par `hdiutil` :**
  `qemu-nbd -c` (un périphérique nbd), les montages FUSE (`fuse2fs`,
  `guestmount`), et une machine virtuelle ou un autre programme qui a l'image
  ouverte.
- Les autres systèmes d'exploitation (voir le tableau) et un périphérique loop
  attaché dans un autre espace de noms de montage.
- Un partage passé en lecture seule à cause d'un montage le reste après un
  rechargement même si le montage a disparu ; redémarrez-le pour l'ouvrir de
  nouveau en écriture.

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
