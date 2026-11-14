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

## HFS+-Images sind schreibgeschützt, solange eine Freigabe nicht zustimmt {#hfsplus-images-are-writable}

Seit v0.44.0 (hfsplus v0.9.0) kann eine Freigabe, deren Image HFS+ enthält, wie
FAT, exFAT und ext4 zum Schreiben geöffnet werden: Ein Client kann Dateien
anlegen, ändern und löschen, und die Zeiten, Modi und Besitzer, die er setzt,
werden gespeichert. Vorher wurde ein HFS+-Image schreibgeschützt geöffnet.

**Seit v0.45.1 standardmäßig aus, und weiterhin aus.** hfsplus v0.9.0 war nicht
absturzsicher (es hielt das ganze Image im Speicher, schrieb es bei jeder
Operation vollständig zurück und markierte das Volume nie als in Benutzung);
deshalb hat v0.45.1 das Schreiben abgeschaltet, das v0.44.0 und v0.45.0
standardmäßig erlaubten (bis 256 MiB). **Seit v0.46.0 (hfsplus v0.10.0)** wird
ein beschreibbares HFS+-Volume **an Ort und Stelle** geschrieben:

- **Kein Speicherbedarf.** Das Image wird nicht in den Speicher kopiert (ein
  Lese-Cache von 8 MiB); nur die Sektoren, die ein Commit ändert, werden
  geschrieben. Die Obergrenze von 64 GiB entfällt.
- **Ein Absturz ist sichtbar.** Jeder Commit markiert das Volume als in
  Benutzung: Das Unmounted-Bit und das Journaled-Bit werden beim ersten Schreiben
  des Headers gelöscht und am Ende wiederhergestellt, sodass `fsck_hfs -q` einen
  Absturz erkennt, auch bei Volumes mit Journal.
- **Ein fehlgeschlagener Schreibvorgang stoppt das Volume.** Nach einem
  fehlgeschlagenen Flush verweigert das Volume Schreibzugriffe
  (`ErrVolumeFailed`), bis die Freigabe neu geöffnet wird.

Es bleibt **auf Zustimmung**, wegen dessen, was hfsplus noch nicht fertig hat:
Ein dauerhaft geschriebener kleiner Schreibvorgang kostet 3 fsyncs pro Commit
und ist etwa 3-mal langsamer als unter macOS (go-filesystems/hfsplus#35), die
Absturztests sind noch nicht gehärtet (go-filesystems/hfsplus#37), und nach einem
Absturz wird ein Volume mit Journal ohne sein Journal eingehängt, bis
`diskutil enableJournal` läuft. Der Standard kann zurückkehren, wenn das behoben
ist. Sichern Sie das Image zuerst. Eine Freigabe stimmt mit
`hfsplus_writable_max` zu, standardmäßig `"0"` (nie; eine Zahl in Bytes oder eine
mit K, M, G oder T):

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"   # zustimmen
}
```

Seit v0.46.0 begrenzt die Größe den Speicher nicht mehr. Sie bleibt eine Größe,
damit eine für v0.44.0 bis v0.45.1 geschriebene Konfiguration bedeutet, was sie
bedeutete (eine Freigabe mit `"256M"` beginnt nicht, ein 2-TiB-Image zu
beschreiben); ein großer Wert wie `"1T"` ist die schlichte Zustimmung.

Beim Standard, über der Größe oder bei unbekannter Größe wird die Freigabe
**schreibgeschützt** geöffnet, weiterhin mit Bereichslesen und Zeiten, und der
Server schreibt eine Zeile mit dem Image und dem Grund, zum Beispiel ohne
Einstellung:

```text
/srv/mac.img is an HFS+ volume and hfsplus_writable_max is 0, the default: HFS+ images are written only when a share sets a size, because a durable small write is about 3x slower than on macOS (go-filesystems/hfsplus#35), the crash tests are not finished (go-filesystems/hfsplus#37), and after a crash a journaled volume mounts without its journal until diskutil enableJournal, so mac is read-only
```

und über der Größe:

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes), so mac is read-only
```

**Ein Neuladen folgt der Einstellung (seit v0.46.0).** Ändert ein Neuladen, ob
ein HFS+-Image geschrieben wird (Einstellung entfernt, hinzugefügt oder über die
Größe des Images hinweg verschoben), wird das Image neu geöffnet: Das Entfernen
der Zustimmung macht die Freigabe sofort schreibgeschützt, das Hinzufügen macht
sie beschreibbar. Vor v0.46.0 blieb der geöffnete Treiber erhalten, sodass eine
Freigabe nach dem Entfernen ihrer Zustimmung bis zum Neustart des Servers
beschreibbar blieb. Der Wechsel von einer beschreibbaren Größe zu einer anderen
ändert nichts.

Ein Volume **mit Journal** (erstellt von `hdiutil create -fs "Journaled HFS+"`
oder von macOS) wird nur geschrieben, wenn sein Journal leer ist, was ein
sauberes Aushängen hinterlässt. Ein Journal mit Transaktionen oder eines, das
sich nicht prüfen lässt, würde von macOS über das wiedergegeben, was der Server
schreibt; der Treiber verweigert es daher, und **die Freigabe wird stattdessen
schreibgeschützt geöffnet**. Der Server startet trotzdem; beim Öffnen der
Freigabe schreibt er eine Zeile mit Image, Freigabe und Grund:

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

Um eine solche Freigabe beschreibbar zu machen, muss das Journal zuerst
wiedergegeben werden: das Image auf einem Mac einhängen und sauber auswerfen
oder `fsck_hfs` darauf ausführen. Solange ein Volume mit Journal geschrieben
wird, ist sein Journaled-Bit gelöscht; nach einem Absturz bleibt es gelöscht,
das Volume wird also ohne Journal eingehängt, bis `diskutil enableJournal`
läuft. Eine Freigabe mit `read_only = true`, ein Gerät und eine Partition
versuchen das Öffnen zum Schreiben nie.

Das Volume wird über den eigenen Dateideskriptor der Freigabe geschrieben
(hfsplus v0.11.0): die Datei, die bereits eingegrenzt und gesperrt ist, nie der
erneut geöffnete Name des Images. Eine Größe, die nicht zum Volume passt, öffnet
die Freigabe schreibgeschützt, mit einer Zeile.

**Ein zweiter Schreiber.** Ein zweiter Schreiber auf demselben Image würde es
beschädigen. Das Öffnen zum Schreiben nimmt eine exklusive beratende Sperre
(`flock`) auf das Image (Linux, macOS und die BSDs); hält ein anderer Prozess
sie, etwa ein zweites fileshare auf demselben Image, wird die Freigabe
schreibgeschützt geöffnet und sagt es. Die Sperre ist beratend: Ein
macOS-Einhängen nimmt sie nicht, und auf anderen Plattformen gibt es keine
Sperre.

⛔ **Schreiben Sie kein eingehängtes Image**, weder mit diesem Server noch mit
etwas anderem: Keine Seite kennt die Änderungen der anderen. Stellen Sie ein
Image bereit, das sonst nichts geöffnet hat.

## Ein beschreibbares Image, das das Betriebssystem eingehängt hat, wird schreibgeschützt bereitgestellt {#an-image-the-os-has-mounted}

Seit v0.45.0 fragt der Server, bevor ein beschreibbares Image-Share (FAT32,
exFAT, ext4, UFS, btrfs, XFS, ZFS, HFS+, unabhängig vom Treiber) zum Schreiben
geöffnet wird, das Betriebssystem, ob es dieses Image angehängt oder eingehängt
hat. Falls ja, wird das Share **schreibgeschützt** geöffnet, entsprechend
markiert, und der Server schreibt eine Zeile mit Image, Gerät und Einhängepunkt
des Betriebssystems sowie dem Share:

```text
/srv/disk.img is in use by the OS (/dev/loop3, mounted at /mnt/disk), and a second writer would corrupt it, so disk is read-only
```

Das oben beschriebene `flock` kann das nicht leisten: Das Betriebssystem nimmt
beim Einhängen eines Images keine Sperre. Das Image wird über Gerät und Inode
erkannt, nie über den Vergleich von Pfadzeichenketten; ein symbolischer oder
harter Link auf die eingehängte Datei wird also ebenfalls erkannt.

| Betriebssystem | Wie gefragt wird |
| --- | --- |
| Linux | `/sys/block/loop*/loop/backing_file` jedes Loop-Geräts (was `mount -o loop`, `losetup` und udisks anlegen); der Einhängepunkt stammt aus `/proc/self/mountinfo`. Ein angehängtes Loop-Gerät zählt auch, wenn nichts darauf eingehängt ist. |
| macOS | `hdiutil info -plist`, der `image-path` jedes angehängten Images, ein Aufruf pro Ladevorgang, mit 15 s Zeitlimit. Ein mit `-nomount` angehängtes Image zählt. |
| FreeBSD, andere BSDs, Windows | **Nicht erkennbar.** Das Share ist wie bisher beschreibbar. |

**Wenn die Prüfung selbst fehlschlägt** (kein `/sys/block` in einem Container,
`hdiutil` fehlt oder überschreitet das Zeitlimit, Ausgabe ist keine Property
List), weiß niemand, ob das Image eingehängt ist. `image_mount_unknown`
entscheidet:

```hcl
share "disk" {
  image               = "/srv/disk.img"
  image_mount_unknown = "read_only"   # der Standard; "write" stellt es trotzdem beschreibbar bereit
}
```

Der Standard ist `read_only`: Ein schreibgeschütztes Share ist ein sichtbarer
Zustand mit einer Logzeile, die die Ursache nennt, und lässt sich beheben; ein
beschreibbares Share unter einem Mount beschädigt die Daten beider Seiten, und
es fällt erst später auf. `write` ist für einen Host, auf dem die Prüfung nicht
funktionieren kann (ein Container ohne sysfs) und auf dem niemand Images
einhängt. Ein Image, das nachweislich vom Betriebssystem eingehängt ist, bleibt
schreibgeschützt, was auch immer hier steht. Ein Backing-Pfad, den die Prüfung
nicht lesen kann (Zugriff verweigert, Symlink-Schleife), gilt als „nicht
beurteilbar“, nicht als „anderes Image“: Er geht an dieselbe Einstellung. Nur
ein nicht vorhandener Pfad wird übersprungen. Unter Linux wird ein Loop-Gerät,
dessen Backing-Datei gelöscht wurde, über den Kernel-Eintrag
(`LOOP_GET_STATUS64`) erkannt, wenn der Prozess `/dev/loopN` öffnen darf, sonst
nicht.

**Was die Prüfung nicht sieht**

- **Ein Mount, der nach dem Öffnen des Images erfolgt.** Die Prüfung läuft, wenn
  ein Image geöffnet wird: beim Start, für ein über die Admin-API erstelltes
  Share und wenn ein Reload die Shares neu aufbaut (Änderung von Personen oder
  Listen) und das Image inzwischen eingehängt findet; dann erhält dieses Share
  eine eigene schreibgeschützte Öffnung. Ein Reload ohne Änderungen prüft nicht
  erneut, und nichts überwacht ein späteres `hdiutil attach` oder
  `mount -o loop`. Halten Sie das Share an oder starten Sie den Server neu,
  bevor Sie ein beschreibbar bereitgestelltes Image einhängen.
- **Mounts ohne Loop-Gerät oder `hdiutil`:** `qemu-nbd -c` (nbd-Gerät),
  FUSE-Mounts (`fuse2fs`, `guestmount`) sowie eine virtuelle Maschine oder ein
  anderes Programm, das das Image geöffnet hält.
- Andere Betriebssysteme (siehe Tabelle) und ein Loop-Gerät, das in einem anderen
  Mount-Namespace angehängt wurde.
- Ein wegen eines Mounts schreibgeschütztes Share bleibt bei einem Reload
  schreibgeschützt, auch wenn der Mount verschwunden ist; starten Sie es neu, um
  es wieder zum Schreiben zu öffnen.

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

## Was SMB mit einer Zeit macht, die es nicht speichern kann {#what-smb-does-with-a-time-it-cannot-store}

Eine Zeit, die ein Client setzt und der Speicher nicht behalten kann, wird
standardmäßig **ignoriert**. Seit v0.42.0 kann eine Freigabe das mit
`unstorable_times` festlegen. Nicht speicherbar ist eine CreationTime, die sich
von der gemeldeten unterscheidet, eine allein gesendete ChangeTime oder eine
Zugriffs- oder Änderungszeit bei einem Image, dessen Treiber keine Zeiten setzen
kann (NTFS, APFS, ISO 9660, SquashFS; ein Host-Verzeichnis und ein FAT-, exFAT-, ext4- oder HFS+-Image
speichern diese):

```hcl
share "photos" {
  directory        = "/data/photos"
  unstorable_times = "log"      # "ignore" (Standard), "refuse" oder "log"
}
```

`refuse` beantwortet die Anfrage des Clients mit einem Fehler, `log` schreibt
eine Zeile mit der Freigabe und dem Pfad. Die genauen Fälle je Art der Freigabe
stehen unter
[Echte Dateizeiten]({{< relref "/protocols/times.md#when-unstorable" >}}).
