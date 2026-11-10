---
title: "Die echten Dateizeiten"
linkTitle: "Dateizeiten"
weight: 60
description: "Welche Freigaben über NFS, SMB, SFTP und WebDAV die eigenen Zeiten jeder Datei melden, was jedes Protokoll sagt, wenn eine Zeit nicht erfasst ist, und was geschieht, wenn ein Client über SMB eine Zeit setzt."
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

## Zeiten, die ein Client über SMB setzt {#setting-times}

Seit v0.41.0 (`go-filesystems/smb` v0.10.0,
go-filesystems/smb#45) werden die Zugriffs- und Änderungszeiten, die ein Client
über SMB setzt (`SET_INFO`, wie es `touch`, `robocopy /COPY:T` und ein Kopieren
im Explorer tun), auf die Datei angewendet, bei einer **Verzeichnis**-Freigabe.
Eine Zeit von 0 lässt diese Zeit unverändert; -1 friert sie ein und -2 hebt das
wieder auf, je geöffnetem Handle.

Eine Zeit, die nicht gespeichert werden kann, wird **standardmäßig ignoriert**:
die Anfrage gelingt und dieses Feld wird verworfen, wie Samba es tut, damit
Kopien, die Zeiten erhalten, nicht scheitern. Das betrifft die Erstellungszeit,
eine allein gesendete Statusänderungszeit und jede Zeit bei einem Treiber, der
keine Zeiten setzen kann. fileshare bietet dafür keine Einstellung an: alle
Freigaben verwenden den Standard.
