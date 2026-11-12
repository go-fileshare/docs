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
keine Zeiten setzen kann.

### Festlegen, was geschieht: `unstorable_times` {#unstorable-times}

Seit v0.42.0 legt eine Freigabe mit `unstorable_times` fest, was mit einer
solchen Zeit geschieht (die Option `UnstorableTimes` von `go-filesystems/smb`):

```hcl
share "attic" {
  directory        = "/srv/attic"
  unstorable_times = "refuse"   # "ignore" (Standard), "refuse" oder "log"
}
```

| Wert | Was der Client hört | Was behalten wird |
|---|---|---|
| `ignore` (auch ohne die Einstellung) | Erfolg | das Feld wird verworfen, wie Samba es tut |
| `refuse` | `STATUS_NOT_SUPPORTED`, damit ein Backup-Werkzeug nie glaubt, eine Zeit sei erhalten geblieben | nichts aus dieser Anfrage wird angewendet |
| `log` | Erfolg | das Feld wird verworfen, und pro Anfrage nennt eine Zeile die Freigabe, den Pfad und die Felder, geschrieben dorthin, wo der Server seine eigenen Zeilen schreibt |

Jeder andere Wert wird beim Lesen der Konfiguration abgelehnt, mit einem Fehler,
der die Freigabe und die drei Wörter nennt. Die Einstellung gehört zu SMB: eine
Freigabe, deren `protocols` SMB auslassen, wird abgelehnt, wenn sie sie setzt.
Einen serverweiten Standard gibt es nicht; sie wird pro Freigabe angegeben. Eine
über die Admin-API angelegte Freigabe verwendet `ignore`.

### Wann eine Zeit nicht gespeichert werden kann {#when-unstorable}

Eine Anfrage wird Feld für Feld beurteilt, und ein nicht speicherbares Feld
genügt: `refuse` lehnt die ganze Anfrage ab, `log` schreibt seine Zeile. Eine
Zeit ist nicht speicherbar, wenn:

1. **CreationTime** auf einen Wert gesetzt wird, der sich von dem unterscheidet,
   den der Treiber meldet (oder der Treiber meldet keinen). Eine Erstellungszeit
   lässt sich nicht setzen. Ein Wert, der dem gemeldeten entspricht, wird
   akzeptiert: es gibt nichts zu ändern.
2. **ChangeTime** **allein** gesendet wird, mit einem Wert, der sich vom
   gemeldeten unterscheidet. Zusammen mit einer Zugriffs- oder Änderungszeit wird
   sie akzeptiert, wenn der Treiber Zeiten setzen kann, weil deren Setzen die
   Statusänderungszeit auf „jetzt" aktualisiert.
3. **Eine Zugriffs- oder Änderungszeit** bei einer Freigabe gesetzt wird, deren
   Treiber keine Zeiten setzen kann (`Chtimes`): jeder echte Wert, ob gleich dem
   aktuellen oder nicht.

Drei Dinge entscheidet die Richtlinie nicht. Eine Zeit von 0 (beibehalten), -1
(einfrieren) oder -2 (fortsetzen) ist nie nicht speicherbar. -1 bei einem
Treiber, der Zeiten setzen, aber nicht melden kann, ergibt in jedem Modus immer
`STATUS_NOT_SUPPORTED`: das ist eine fehlende Fähigkeit, kein verworfenes Feld
(betrifft ufs-, btrfs-, xfs- und zfs-Images). Und eine Anfrage an eine
**schreibgeschützte** Freigabe (`read_only = true`, ein Gerät, eine Freigabe mit
gewählter Partition, eine Person ohne Schreibrecht) wird in jedem Modus vor all
dem mit `STATUS_MEDIA_WRITE_PROTECTED` beantwortet.

Was eine Freigabe speichern kann, hängt von ihrem Treiber ab, in den Versionen,
die dieses Release festschreibt:

| Freigabe | CreationTime | ChangeTime allein | Zugriffs- oder Änderungszeit |
|---|---|---|---|
| **Verzeichnis** (Host) | nicht speicherbar, außer sie entspricht der gemeldeten Erstellungszeit | nicht speicherbar, außer sie entspricht der gemeldeten | gespeichert (eine mitgesendete ChangeTime wird akzeptiert) |
| **Image, Treiber mit `Chtimes`**: ext4, ufs, btrfs, xfs, zfs | nicht speicherbar, außer sie entspricht der gemeldeten (nur ext4 meldet eine: bei den anderen immer) | ebenso | gespeichert (eine mitgesendete ChangeTime wird akzeptiert) |
| **Image, FAT oder exFAT** (seit v0.43.0) | nicht speicherbar, außer sie entspricht der gemeldeten (beide melden eine) | ebenso | gespeichert in der Auflösung des Formats: FAT-Schreibzeit 2 s (abgerundet) und Zugriffszeit nur das Datum; exFAT-Schreibzeit 10 ms und Zugriffszeit 2 s (eine mitgesendete ChangeTime wird akzeptiert) |
| **Image, HFS+** (seit v0.44.0) | nicht speicherbar, außer sie entspricht der gemeldeten (es meldet eine) | ebenso | sekundengenau gespeichert (eine mitgesendete ChangeTime wird akzeptiert), wenn die Freigabe `hfsplus_writable_max` setzt (seit v0.45.1; standardmäßig ist sie schreibgeschützt), und nicht, wenn das Journal die Freigabe [schreibgeschützt]({{< relref "/configuration/shares.md#hfsplus-images-are-writable" >}}) gemacht |
| **Image, Treiber ohne `Chtimes`**: NTFS, APFS, ISO 9660, SquashFS | ebenso (NTFS meldet eine; die anderen nie) | ebenso, **auch** wenn sie mit einer Zugriffs- oder Änderungszeit gesendet wird | **immer nicht speicherbar** |

Treiber mit `Chtimes` sind osfs (`osfs.go:383`), ext4 (`metadata.go:93`), ufs
(`metadata.go:63`), btrfs (`fs.go:382`), xfs (`fs.go:521`) und zfs
(`metadata.go:94`) sowie seit v0.43.0 fat32 v0.13.0 und exFAT v0.6.0 (der vollständige
`MetadataSetter`: ihre `Chmod` und `Chown` melden „nicht unterstützt“, da FAT weder
Modus-Bits noch Eigentümer kennt), sodass `touch -d` und `cp -p` auf einer FAT- oder
exFAT-Image-Freigabe gespeichert werden; die übrigen haben keines. Auch HFS+ hat eines:
hfsplus v0.8.0 implementiert `Chtimes`, `Chmod` und `Chown`, und seit v0.44.0
(hfsplus v0.9.0) kann ein bereitgestelltes HFS+-Image schreibbar geöffnet werden, sodass
eine per SMB gesetzte Zeit gespeichert wird, seit v0.45.1 aber nur, wenn die
Freigabe `hfsplus_writable_max` setzt (standardmäßig ist sie schreibgeschützt). Ein Image mit Journal, das
Transaktionen enthält, wird stattdessen schreibgeschützt geöffnet und speichert
nichts (siehe [HFS+-Images sind schreibbar]({{< relref "/configuration/shares.md#hfsplus-images-are-writable" >}})). Vor v0.42.0 verbarg die Sperre, die
fileshare vor jeden Image-Treiber setzt, `Chtimes`, sodass jedes Image die
Zugriffs- und Änderungszeiten eines Clients verwarf, auch ext4; seit v0.42.0
reicht sie `Chtimes` genau dann durch, wenn der Treiber es hat. NTFS-Images liest
ein schreibgeschützter Treiber, daher wird unabhängig von der Richtlinie nichts
gespeichert; die Richtlinie bestimmt nur, was der Client erfährt.

Welche Clients welchen Fall auslösen: Windows-Explorer und `robocopy /COPY:T`
senden eine CreationTime, die keine Freigabe behalten kann, außer sie ist
unverändert; Linux `touch -d` und `cp -p` senden nur Zugriffs- und
Änderungszeiten (der Kernel-Client fügt eine ChangeTime „jetzt" hinzu), die ein
Verzeichnis und ein Treiber mit `Chtimes` speichern, ein Treiber ohne es nicht.
