---
title: "Die echten Dateizeiten"
linkTitle: "Dateizeiten"
weight: 60
description: "Welche Freigaben über NFS, SMB, SFTP und WebDAV die eigenen Zeiten jeder Datei melden, was jedes Protokoll sagt, wenn eine Zeit nicht erfasst ist, und was noch nicht unterstützt wird."
tags: [protokolle, zeitstempel]
---

Seit v0.39.0 sind die Zeiten einer Datei die, die ihr Dateisystem erfasst hat,
und nicht der Zeitpunkt der Anfrage. Das setzt die optionale Fähigkeit
`TimesReader` aus `go-filesystems/interface` v0.6.0 voraus, die ein Treiber
hat oder nicht hat.

## Welche Freigaben echte Zeiten melden {#which-shares-report-real-times}

| Freigabe | Echte Zeiten |
|---|---|
| **Verzeichnis** (Host) | ja: Zugriff, Änderung, Statusänderung und, wo der Host sie führt, Erstellung, auf die Nanosekunde |
| **FAT-Abbild** | ja, in der Auflösung von FAT: Änderung 2 s, Zugriff ein Tag, Erstellung 10 ms; keine Statusänderungszeit |
| **exFAT-Abbild** (seit v0.40.0) | ja: Erstellung und Änderung 10 ms, Zugriff 2 s; keine Statusänderungszeit |
| **NTFS-Abbild** (seit v0.40.0) | ja: Erstellung, Änderung, Statusänderung und Zugriff, 100 ns |
| **ext4-Abbild** (seit v0.40.0) | ja: Zugriff, Änderung und Statusänderung auf 1 ns (1 s bei 128-Byte-Inodes von ext2/ext3); Erstellung nur, wenn der Inode groß genug ist |
| **HFS+-Abbild** (seit v0.40.0) | ja: Erstellung, Änderung, Statusänderung und Zugriff, 1 s |
| Andere Abbildtypen | Antwort wie bisher |
| Als die Person bereitgestellte Freigabe (`remote`-Worker) | ja, wenn das Verzeichnis des Workers ein Host-Verzeichnis ist: die Zeit überquert die Grenze des Workers |

Eine Zeit wird in der **gröberen** der Auflösung des Formats und der des
Protokolls verglichen: SFTP überträgt ganze Sekunden, ein WebDAV-
`getlastmodified` ebenso, SMB 100 ns, NFS 1 ns.

## Wenn eine Zeit nicht erfasst ist {#when-a-time-is-not-recorded}

| Protokoll | Was gesendet wird |
|---|---|
| **SMB** | 0, „nicht angegeben". Ein Treiber **ohne** die Fähigkeit antwortet weiterhin mit der aktuellen Zeit. |
| **NFS** | Eine fehlende Zugriffs- oder Statusänderungszeit wird zur Änderungszeit; eine fehlende Änderungszeit wird zur Startzeit des Servers. |
| **SFTP** | Eine fehlende Änderungszeit löscht das Flag `ACMODTIME`: der Client sieht keine Zeit statt einer falschen. |
| **WebDAV** | `creationdate` ist die Erstellungszeit; eine nicht erfasste Zeit fehlt in der Antwort. |

## Noch nicht unterstützt {#not-supported-yet}

Ein Client, der über SMB eine Zeit **setzt** (`SET_INFO`), wird nicht
befolgt: die Zeit bleibt die, die das Dateisystem erfasst hat
(go-filesystems/smb#45).
