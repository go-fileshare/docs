---
title: "Freigaben: Images, Geräte, Verzeichnisse"
weight: 10
description: "Was eine Freigabe bereitstellen kann (ein Disk-Image, ein Gerät oder ein Verzeichnis des Hosts) und wie ihr Dateisystem und ihre Partition gewählt werden."
tags: [konfiguration, freigaben]
---

Eine Freigabe stellt eines von drei Dingen bereit: ein **Disk-Image**, einen **Geräteknoten** oder ein
**Verzeichnis des Hosts**. Ein Image und ein Gerät tragen ein Dateisystem, das dieses
Programm selbst liest; ein Verzeichnis ist das des Hosts.

## Ein Image und das Dateisystem darin {#an-image-and-the-filesystem-inside-it}

Das Dateisystem in einem Image wird **ermittelt statt deklariert**.
[`go-filesystems/detect`](https://github.com/go-filesystems/detect) liest die
Magic-Bytes und gibt den Treiber zurück, dem sie gehören: **fat32, exfat, ext4, ntfs, ufs,
iso9660, squashfs oder hfsplus** – jeder Treiber derselben Form,
`OpenReader(io.ReaderAt, int64)`, ein Dateisystem bei Offset null.

## Die vier, die sich nicht erschnüffeln lassen {#the-four-that-cannot-be-sniffed}

**apfs, btrfs, xfs und zfs** öffnen jeweils ein *Disk*-Image und wählen eine **Partition**,
also gibt es bei Offset null keine Magic-Bytes zu finden. Eine Freigabe sagt, welches:

```hcl
share "photos" {
  image      = "/srv/disk.img"
  filesystem = "xfs"
  partition  = 2       # or leave it out: the driver takes the first data partition
}
```

## Eine Partition, für jedes Dateisystem {#a-partition-for-any-filesystem}

Ein Disk-Image mit FAT32, ext4 oder exFAT hat ebenso oft eine Partitionstabelle davor
wie eines mit XFS – und die Erkennung liest Offset null, wo ein
partitioniertes Image die *Tabelle* hat. Also wird zuerst die Partition gewählt, und der
Treiber erhält eine Sicht darauf:

```hcl
share "photos" {
  image           = "/srv/disk.img"
  partition_label = "photos"          # what lsblk calls PARTLABEL
}
```

Drei Arten, eine zu benennen, und **nur eine darf angegeben werden** – zwei, die sich widersprechen, würden
diejenige bereitstellen, die der Code zuerst versucht hat:

| | |
|---|---|
| `partition = 2` | ab 1 gezählt, so wie Partitionierungswerkzeuge sie ausgeben |
| `partition_label = "photos"` | der GPT-Partitionsname |
| `partition_uuid = "…"` | die eindeutige GPT-GUID – Linux' `PARTUUID` |

{{< callout type="error" >}}
**Ein Index verschiebt sich**

Eine neu partitionierte Platte, ein Werkzeug, das Einträge in anderer Reihenfolge schreibt, ein Image,
das mit einer Partition weniger wiederhergestellt wurde – und `partition = 2` benennt etwas
anderes, **stillschweigend**, weil dort immer noch ein Dateisystem gefunden wird. Ein Label oder eine
UUID benennt die Partition selbst; deshalb verwenden fstabs seit Jahren keine Indizes
mehr. Der Index ist für MBR-Images da, die weder das eine noch das andere haben.
{{< /callout >}}

Eine Freigabe, die eine Partition gewählt hat, ist **schreibgeschützt** und sagt das beim Start: Die
Offsets des Treibers sind die der Partition, während die Datei darunter die ganze
Platte ist, sodass ein Schreibzugriff bei diesem Offset vom Anfang des *Images* aus landen würde – nicht selten
auf der Partitionstabelle.

## Ein Dateisystem zu benennen schaltet die Erkennung ab {#naming-a-filesystem-turns-detection-off}

{{< callout type="error" >}}
**Das ist der Zweck, und das ist das Risiko**

Das Image wird als *genau das* geöffnet oder abgelehnt. Ein FAT32-Image, dem gesagt wird, es sei XFS,
wird nicht zu einer XFS-Freigabe; es startet nicht und sagt, als was es
geöffnet werden sollte.
{{< /callout >}}

`filesystem` wird auch für die erkennbaren akzeptiert, und dann werden beide
verglichen: Eine Freigabe, die `ext4` über einem FAT32-Image angibt, wird mit *the
share says ext4 and the image holds fat32* abgelehnt. So lehnt eine Installation eine
Fehlerkennung ab, statt sie später zu entdecken.
[`check`]({{< relref "/configuration/check.md" >}}) markiert einen benannten Treiber mit einem Sternchen, weil diese Zeile nicht
erkannt wurde – sie wurde festgelegt.

## Ein Gerät, nicht nur ein Image {#a-device-not-only-an-image}

```hcl
share "photos" {
  image = "/dev/sda"        # a device node, not a file
}
```

**Kein Privileg ist im Spiel.** Hier wird nichts eingehängt – die
[`go-filesystems`](https://github.com/go-filesystems)-Treiber lesen ext4, xfs und
die übrigen im Userspace –, also gibt es kein `mount(2)` und daher kein
`CAP_SYS_ADMIN`. Ein Gerät zu öffnen ist ein gewöhnliches `open(2)`, geregelt durch die
Berechtigungen des Knotens:

| | Eigentümer | Modus | genügt |
|---|---|---|---|
| Linux `/dev/sda` | `root:disk` | 0660 | Mitgliedschaft in `disk` |
| macOS `/dev/disk0` | `root:operator` | 0640 | Mitgliedschaft in `operator`, plus Festplattenvollzugriff |

Eine Ablehnung nennt die Gruppe, statt `permission denied` – die Antwort ist
nie `sudo`.

⛔ **Eine Gerätefreigabe ist schreibgeschützt**, aus demselben Grund wie eine Freigabe, die eine
Partition gewählt hat: In eine laufende Platte zu schreiben ist keine Entscheidung, die man für Sie trifft.

{{< callout type="error" >}}
**Das Gerät wird exklusiv geöffnet, und dabei geht es um Korrektheit, nicht um Vorsicht**

Hat der Kernel ein Dateisystem davon eingehängt, hält der Page Cache des Dateisystems
neuere Metadaten als das Gerät – sodass ein roher Leser einen
Verzeichnisblock von vor einer Aktualisierung neben einem Inode-Block von danach sieht,
einen Zustand, der zu keinem Zeitpunkt auf der Platte existiert hat. `O_EXCL` auf einem Block-Device
ist das eigene Primitiv des Kernels für „niemand sonst, auch kein Mount";
es ist das, was `mkfs` und `fsck` verwenden. Ein belegtes Gerät wird abgelehnt, mit Namen und
Grund.
{{< /callout >}}

Zwei Dinge wurden gemessen statt angenommen:

- `Stat().Size()` ist für ein Gerät **0**, also wird die Länge beim Gerät
  selbst erfragt. Ein Seek ans Ende antwortet unter Linux und liefert unter macOS **0 ohne Fehler**,
  weshalb Darwin stattdessen `DKIOCGETBLOCKCOUNT` verwendet.
- Ein **roher** Knoten (`/dev/rdiskN`, oder `O_DIRECT` unter Linux) lehnt einen nicht ausgerichteten
  Lesezugriff ab, und ein Zwei-Byte-Feld bei Offset 11 zu lesen ist genau das, was das Parsen eines FAT-BPB
  *ist*. Lesezugriffe auf ein Gerät werden auf ganze Blöcke aufgerundet; ohne das
  meldete `/dev/rdisk4` `unknown filesystem`.

## Ein Verzeichnis, nicht nur ein Image {#a-directory-not-only-an-image}

Seit v0.8.0 kann eine Freigabe statt eines Images ein Verzeichnis des Hosts bereitstellen – etwa das Volume
eines Containers:

```hcl
share "photos" {
  directory = "/data/photos"
  allow     = ["@family"]
}
```

Es ist [go-filesystems/osfs](https://github.com/go-filesystems/osfs), das
den Baum über ein **`os.Root`** erreicht: `..`, das hinausklettert, ein absoluter
Pfad und ein symbolischer Link, der hinausführt, werden von der kernelgestützten Wurzel abgelehnt,
nicht durch einen String-Vergleich. Ein Client darf einen Link auf `/etc` *anlegen*; nichts wird
ihm folgen. Ein im Baum platziertes FIFO wird abgelehnt, statt den Server
hängen zu lassen.

{{< callout type="error" >}}
**Eine Freigabe darf keine andere enthalten (seit v0.17.0)**

Eine Verzeichnisfreigabe, deren Baum das Image oder Verzeichnis einer anderen Freigabe enthält, wird
**beim Start abgelehnt**, und auch von der Admin-API: Wer die äußere
Freigabe nutzen darf, würde die innere lesen und schreiben, ohne dass es ihm erlaubt ist –
anonymes NFS eingeschlossen. Ebenso zwei Freigaben auf derselben Quelle, außer zwei
verschiedenen Partitionen eines Disk-Images.
{{< /callout >}}

{{< callout type="info" >}}
**Nicht hinter der Image-Sperre**

Ein Image-Treiber besitzt eine Datei und sichert nichts über zwei gleichzeitige Aufrufe zu,
weshalb [jedes Image in eine Sperre gehüllt wird]({{< relref "/operations/locking.md" >}}).
Ein Baum des Hosts gehört dem Kernel, der serialisiert, was pro Aufruf serialisiert werden muss,
und nicht mehr; ein Mutex vor jeder Datei jedes Clients wäre
das Langsamste, was ein Dateiserver tun kann.
{{< /callout >}}

⛔ Eine Freigabe hat ein `image` **oder** ein `directory`, nie beides und nie keines von beiden.
`filesystem` und `partition` sind für ein Image: Eine Verzeichnisfreigabe, die eines davon nennt, wird
abgelehnt, weil die Angabe bedeutet, dass die Freigabe als Image gedacht war.
`read_only = true` funktioniert bei einem Verzeichnis wie bei einem Image; eine Freigabe, die das sagt
und zugleich `writers` auflistet, wird abgelehnt, weil `read_only` gewinnen würde.

Die Kapazität, die einem Client mitgeteilt wird, ist die des Dateisystems, auf dem der Baum liegt; eine
Plattform, die das nicht angeben kann, lässt sie bei null, was die Protokolle als
„unbekannt" lesen statt als „voll".
