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

## Welche Freigaben echte Zeiten melden

| Freigabe | Echte Zeiten |
|---|---|
| **Verzeichnis** (Host) | ja: Zugriff, Änderung, Statusänderung und, wo der Host sie führt, Erstellung, auf die Nanosekunde |
| **FAT-Abbild** | ja, in der Auflösung von FAT: Änderung 2 s, Zugriff ein Tag, Erstellung 10 ms; keine Statusänderungszeit |
| exFAT-, NTFS-, ext4-, HFS+-Abbilder | noch nicht (exfat#29, ntfs#32, ext4#41, hfsplus#25): Antwort wie bisher |
| Andere Abbildtypen | Antwort wie bisher |
| Als die Person bereitgestellte Freigabe (`remote`-Worker) | ja, wenn das Verzeichnis des Workers ein Host-Verzeichnis ist: die Zeit überquert die Grenze des Workers |

Eine Zeit wird in der **gröberen** der Auflösung des Formats und der des
Protokolls verglichen: SFTP überträgt ganze Sekunden, ein WebDAV-
`getlastmodified` ebenso, SMB 100 ns, NFS 1 ns.

## Wenn eine Zeit nicht erfasst ist

| Protokoll | Was gesendet wird |
|---|---|
| **SMB** | 0, „nicht angegeben". Ein Treiber **ohne** die Fähigkeit antwortet weiterhin mit der aktuellen Zeit. |
| **NFS** | Eine fehlende Zugriffs- oder Statusänderungszeit wird zur Änderungszeit; eine fehlende Änderungszeit wird zur Startzeit des Servers. |
| **SFTP** | Eine fehlende Änderungszeit löscht das Flag `ACMODTIME`: der Client sieht keine Zeit statt einer falschen. |
| **WebDAV** | `creationdate` ist die Erstellungszeit; eine nicht erfasste Zeit fehlt in der Antwort. |

## Noch nicht unterstützt

Ein Client, der über SMB eine Zeit **setzt** (`SET_INFO`), wird nicht
befolgt: die Zeit bleibt die, die das Dateisystem erfasst hat
(go-filesystems/smb#45).
