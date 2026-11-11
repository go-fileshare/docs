---
title: "Les vraies dates des fichiers"
linkTitle: "Dates des fichiers"
weight: 60
description: "Quels partages indiquent les dates propres à chaque fichier en NFS, SMB, SFTP et WebDAV, ce que dit chaque protocole quand une date n'est pas enregistrée, et ce qui se passe quand un client fixe une date en SMB."
tags: [protocoles, dates]
---

Depuis la v0.39.0, les dates d'un fichier sont celles que son système de
fichiers a enregistrées, et non l'heure de la requête. Cela demande la
capacité optionnelle `TimesReader` de `go-filesystems/interface` v0.6.0, qu'un
pilote a ou n'a pas.

## Quels partages indiquent les vraies dates {#which-shares-report-real-times}

| Partage | Vraies dates |
|---|---|
| **Répertoire** (hôte) | oui : accès, modification, changement et, si l'hôte la garde, création, à la nanoseconde |
| **Image FAT** | oui, à la résolution de FAT : modification 2 s, accès un jour, création 10 ms ; pas de date de changement |
| **Image exFAT** (depuis v0.40.0) | oui : création et modification 10 ms, accès 2 s ; pas de date de changement |
| **Image NTFS** (depuis v0.40.0) | oui : création, modification, changement et accès, 100 ns |
| **Image ext4** (depuis v0.40.0) | oui : accès, modification et changement à 1 ns (1 s sur les inodes ext2/ext3 de 128 octets) ; création seulement si l'inode est assez grand |
| **Image HFS+** (depuis v0.40.0) | oui : création, modification, changement et accès, 1 s |
| Autres types d'image | répondu comme avant |
| Partage servi sous l'identité de la personne (worker `remote`) | oui, si le répertoire du worker est un répertoire de l'hôte : la date traverse la frontière du worker |

Une date est comparée à la résolution la **plus grossière** entre celle du
format et celle du protocole : SFTP transporte des secondes entières, le
`getlastmodified` de WebDAV aussi, SMB 100 ns, NFS 1 ns.

## Quand une date n'est pas enregistrée {#when-a-time-is-not-recorded}

| Protocole | Ce qui est envoyé |
|---|---|
| **SMB** | 0, « non spécifié ». Un pilote **sans** la capacité répond toujours l'heure courante. |
| **NFS** | Une date d'accès ou de changement manquante devient la date de modification ; une date de modification manquante devient l'heure de démarrage du serveur. |
| **SFTP** | Une date de modification manquante efface le drapeau `ACMODTIME` : le client ne voit aucune date plutôt qu'une fausse. |
| **WebDAV** | `creationdate` est la date de création ; une date non enregistrée est absente de la réponse. |

## Dates fixées par un client en SMB {#setting-times}

Depuis la v0.41.0 (`go-filesystems/smb` v0.10.0,
go-filesystems/smb#45), les dates d'accès et de modification qu'un client fixe
en SMB (`SET_INFO`, comme le font `touch`, `robocopy /COPY:T` et une copie dans
l'Explorateur) sont appliquées au fichier, sur un partage de **répertoire**. Une
date à 0 la laisse telle quelle ; -1 la fige et -2 la libère, par descripteur
ouvert.

Une date qui ne peut pas être enregistrée est **ignorée par défaut** : la
requête réussit et ce champ est abandonné, comme le fait Samba, afin que les
copies qui conservent les dates n'échouent pas. Cela vise la date de création,
une date de changement envoyée seule, et toute date sur un pilote qui ne sait
pas fixer les dates.

### Choisir ce qui se passe : `unstorable_times` {#unstorable-times}

Depuis la v0.42.0, un partage dit ce qu'il advient d'une telle date avec
`unstorable_times` (l'option `UnstorableTimes` de `go-filesystems/smb`) :

```hcl
share "attic" {
  directory        = "/srv/attic"
  unstorable_times = "refuse"   # "ignore" (par défaut), "refuse" ou "log"
}
```

| Valeur | Ce qu'entend le client | Ce qui est conservé |
|---|---|---|
| `ignore` (aussi quand le réglage est absent) | succès | le champ est abandonné, comme le fait Samba |
| `refuse` | `STATUS_NOT_SUPPORTED`, pour qu'un outil de sauvegarde ne croie jamais qu'une date a été conservée | rien de cette requête n'est appliqué |
| `log` | succès | le champ est abandonné, et une ligne par requête nomme le partage, le chemin et les champs, écrite là où le serveur écrit ses propres lignes |

Toute autre valeur est refusée à la lecture de la configuration, par une erreur
qui nomme le partage et les trois mots. Le réglage appartient à SMB : un partage
dont `protocols` exclut SMB est refusé s'il le fixe. Il n'y a pas de valeur par
défaut pour tout le serveur ; on la dit sur chaque partage. Un partage créé par
l'API d'administration utilise `ignore`.

### Quand une date ne peut pas être enregistrée {#when-unstorable}

Une requête est jugée champ par champ, et un seul champ non enregistrable
suffit : `refuse` refuse toute la requête, `log` écrit sa ligne. Une date n'est
pas enregistrable quand :

1. **CreationTime** est fixée à une valeur différente de celle que le pilote
   annonce (ou le pilote n'en annonce aucune). Il n'existe aucun moyen de fixer
   une date de création. Une valeur égale à celle annoncée est acceptée : il n'y
   a rien à changer.
2. **ChangeTime** est envoyée **seule**, avec une valeur différente de celle
   annoncée. Envoyée avec une date d'accès ou d'écriture, elle est acceptée si
   le pilote sait fixer les dates, parce que les fixer remet la date de
   changement à « maintenant ».
3. **Une date d'accès ou d'écriture** est fixée sur un partage dont le pilote ne
   sait pas fixer les dates (`Chtimes`) : toute valeur réelle, égale à la
   courante ou non.

Trois choses ne sont pas décidées par la politique. Une date de 0 (conserver),
-1 (figer) ou -2 (reprendre) n'est jamais non enregistrable. -1 sur un pilote qui
sait fixer les dates mais pas les annoncer donne toujours
`STATUS_NOT_SUPPORTED`, dans tous les modes : c'est une capacité absente, pas un
champ abandonné (cela concerne les images ufs, btrfs, xfs et zfs). Et une
requête sur un partage **en lecture seule** (`read_only = true`, un périphérique,
un partage qui a choisi une partition, une personne sans droit d'écriture)
reçoit `STATUS_MEDIA_WRITE_PROTECTED` avant tout cela, dans tous les modes.

Ce qu'un partage peut enregistrer dépend de son pilote, dans les versions
qu'épingle cette version :

| Partage | CreationTime | ChangeTime seule | Date d'accès ou d'écriture |
|---|---|---|---|
| **Répertoire** (hôte) | non enregistrable sauf si égale à la date de création annoncée | non enregistrable sauf si égale à l'annoncée | enregistrée (une ChangeTime envoyée avec elle est acceptée) |
| **Image, pilote avec `Chtimes`** : ext4, ufs, btrfs, xfs, zfs | non enregistrable sauf si égale à l'annoncée (seul ext4 en annonce une : pour les autres, toujours) | idem | enregistrée (une ChangeTime envoyée avec elle est acceptée) |
| **Image, FAT ou exFAT** (depuis la v0.43.0) | non enregistrable sauf si égale à l'annoncée (les deux en annoncent une) | idem | enregistrée à la résolution du format : date d'écriture FAT à 2 s (arrondie vers le bas) et date d'accès au jour seulement ; date d'écriture exFAT à 10 ms et date d'accès à 2 s (une ChangeTime envoyée avec elle est acceptée) |
| **Image, pilote sans `Chtimes`** : NTFS, HFS+, APFS, ISO 9660, SquashFS | idem (NTFS et HFS+ en annoncent une ; les autres jamais) | idem, **y compris** envoyée avec une date d'accès ou d'écriture | **toujours non enregistrable** |

Les pilotes avec `Chtimes` sont osfs (`osfs.go:383`), ext4 (`metadata.go:93`), ufs
(`metadata.go:63`), btrfs (`fs.go:382`), xfs (`fs.go:521`) et zfs
(`metadata.go:94`) et, depuis la v0.43.0, fat32 v0.13.0 et exFAT v0.6.0 (le
`MetadataSetter` complet : leurs `Chmod` et `Chown` répondent « non pris en charge »,
FAT n'ayant ni bits de mode ni propriétaires), si bien que `touch -d` et `cp -p` sur un
partage d'image FAT ou exFAT sont enregistrés ; les autres n'en ont pas. HFS+ reste dans
cette liste : hfsplus v0.8.0 (épinglé depuis la v0.43.0) implémente `Chtimes`, `Chmod`
et `Chown`, mais une image HFS+ servie est ouverte par le `OpenReader` en lecture seule
du pilote, si bien qu'une date fixée par SMB n'y est toujours pas enregistrée. Ce que le
pilote écrit dans une image HFS+ créée par `hdiutil` avec journal n'est pas encore vu par
macOS (Refs go-filesystems/hfsplus#29). Avant la v0.42.0, le verrou que
fileshare place devant chaque pilote d'image masquait `Chtimes`, si bien que
toute image abandonnait les dates d'accès et d'écriture d'un client, ext4
comprise ; depuis la v0.42.0 il le transmet exactement quand le pilote en a un.
Les images NTFS sont lues par un pilote en lecture seule : rien n'est enregistré
quelle que soit la politique, qui décide seulement de ce que le client apprend.

Quels clients déclenchent quel cas : l'Explorateur Windows et `robocopy /COPY:T`
envoient une CreationTime, qu'un partage de n'importe quel type ne peut pas
conserver si elle change ; `touch -d` et `cp -p` sous Linux n'envoient que des
dates d'accès et d'écriture (le client du noyau ajoute une ChangeTime
« maintenant »), qu'un répertoire et un pilote avec `Chtimes` enregistrent, et
un pilote sans lui non.
