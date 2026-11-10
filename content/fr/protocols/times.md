---
title: "Les vraies dates des fichiers"
linkTitle: "Dates des fichiers"
weight: 60
description: "Quels partages indiquent les dates propres à chaque fichier en NFS, SMB, SFTP et WebDAV, ce que dit chaque protocole quand une date n'est pas enregistrée, et ce qui n'est pas encore pris en charge."
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

## Pas encore pris en charge {#not-supported-yet}

Un client qui **modifie** une date en SMB (`SET_INFO`) n'est pas obéi : la date
reste celle que le système de fichiers avait enregistrée
(go-filesystems/smb#45).
