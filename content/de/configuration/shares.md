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

Seit v0.44.0 (hfsplus v0.9.0) wird eine Freigabe, deren Image HFS+ enthält, zum
Schreiben geöffnet, wie FAT, exFAT und ext4: Ein Client kann Dateien anlegen,
ändern und löschen, und die Zeiten, Modi und Eigentümer, die er setzt, werden
gespeichert. Vorher wurde ein HFS+-Image schreibgeschützt geöffnet. Das Image
wird beim Öffnen der Freigabe in den Speicher gelesen, und jede Änderung wird in
die Datei zurückgeschrieben.

**Seit v0.45.1 standardmäßig aus.** hfsplus v0.9.0 ist nicht absturzsicher: Es
schreibt bei jeder Operation das ganze Image neu und markiert das Volume dabei
nie als in Benutzung; ein Absturz oder Stromausfall mitten im Schreiben
hinterlässt ein zerrissenes Image, das weiterhin sauber ausgehängt zu sein
behauptet, und beim nächsten Einhängen prüft es nichts. Bei einem Volume mit
Journal braucht die Lösung mehr als Schreiben an Ort und Stelle. v0.44.0 und
v0.45.0 schrieben standardmäßig (bis 256 MiB); seit v0.45.1 wird eine HFS+-Freigabe
**schreibgeschützt** bereitgestellt, solange sie nicht `hfsplus_writable_max`
setzt. Die Lösung ist go-filesystems/hfsplus#34; der Standard kann zurückkehren,
wenn sie veröffentlicht ist. Zum Einwilligen eine Größe setzen (und das Image
vorher sichern) und dieses Risiko sowie die Kosten unten in Kauf nehmen.

**Was das kostet.** hfsplus v0.9.0 hält das ganze Image im Speicher und schreibt
bei jeder Operation (Anlegen, Schreiben, Umbenennen, `Chtimes`) das ganze Image
neu; der Speicher einer Freigabe ist also die Größe ihres Images, und ebenso der
E/A-Aufwand jeder Änderung (go-filesystems/hfsplus#33 ist die eigentliche
Lösung). Eine Freigabe öffnet ein HFS+-Image deshalb nur bis zu
`hfsplus_writable_max` zum Schreiben, standardmäßig `"0"` (nie; eine
Anzahl Bytes oder eine mit K, M, G oder T):

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"   # einwilligen
}
```

Darüber, oder wenn die Größe unbekannt ist, wird die Freigabe **schreibgeschützt**
geöffnet, weiterhin mit Bereichslesen und Zeiten, und der Server schreibt eine
Zeile mit Image und Grund, zum Beispiel ohne Einstellung:

```text
/srv/mac.img is an HFS+ volume and hfsplus_writable_max is 0, the default: hfsplus v0.9.0 is not crash-safe (a crash mid-write can leave a torn image marked clean), so HFS+ images are written only when a share sets a size, so mac is read-only
```

und über der Größe:

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes): a writable HFS+ image is held in memory and rewritten whole on every change, so mac is read-only
```

Ein Volume **mit Journal** (erzeugt von `hdiutil create -fs "Journaled HFS+"`
oder von macOS) wird nur beschrieben, wenn sein Journal leer ist, wie es nach
einem sauberen Aushängen zurückbleibt. Ein Journal mit Transaktionen oder eines,
das sich nicht prüfen lässt, würde von macOS über das wiedergegeben, was der
Server schreibt; deshalb lehnt der Treiber es ab, und **die Freigabe wird
stattdessen schreibgeschützt geöffnet**. Der Server startet trotzdem; beim
Öffnen der Freigabe schreibt er eine Zeile mit Image, Freigabe und Grund:

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

Damit eine solche Freigabe schreibbar wird, muss das Journal zuerst
wiedergegeben werden: das Image an einem Mac einhängen und sauber auswerfen
oder `fsck_hfs` darauf ausführen. (Der Treiber hat keine Journalprüfung ohne das
Image im Speicher; dieser eine Fall liest das Image bis zur Grenze, bevor er
zurückfällt.) Eine Freigabe mit `read_only = true`, ein Gerät und eine Partition
versuchen das schreibbare Öffnen nie.

**Ein anderer Schreiber.** Das Image wird ganz zurückgeschrieben, ein zweiter
Schreiber würde also vom nächsten Zurückschreiben überschrieben. Das schreibbare
Öffnen nimmt eine exklusive beratende `flock`-Sperre auf das Image (Linux, macOS
und die BSDs); hält sie ein anderer Prozess, etwa ein zweites fileshare auf
demselben Image, wird die Freigabe schreibgeschützt geöffnet und sagt es. Die
Sperre ist beratend: ein macOS-Einhängen nimmt sie nicht, und auf anderen
Plattformen gibt es keine Sperre.

⛔ **Schreiben Sie kein Image, das eingehängt ist**, weder mit diesem Server
noch mit etwas anderem: keine Seite weiß von den Änderungen der anderen.
Stellen Sie ein Image bereit, das sonst nichts geöffnet hat.

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
