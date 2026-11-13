---
title: "Shares: images, devices, directories"
weight: 10
description: "What a share can serve (a disk image, a device or a host directory) and how its filesystem and partition are chosen."
tags: [configuration, shares]
---

A share serves one of three things: a **disk image**, a **device** node, or a
**directory of the host**. An image and a device carry a filesystem this
program reads itself; a directory is the host's own.

## An image, and the filesystem inside it

The filesystem inside an image is **worked out rather than declared**.
[`go-filesystems/detect`](https://github.com/go-filesystems/detect) reads the
magic and hands back the driver that owns it: **fat32, exfat, ext4, ntfs, ufs,
iso9660, squashfs or hfsplus** — every driver of the one shape,
`OpenReader(io.ReaderAt, int64)`, a filesystem at offset zero.

## The four that cannot be sniffed

**apfs, btrfs, xfs and zfs** each open a *disk* image and pick a **partition**,
so there is no magic at offset zero to find. A share says which:

```hcl
share "photos" {
  image      = "/srv/disk.img"
  filesystem = "xfs"
  partition  = 2       # or leave it out: the driver takes the first data partition
}
```

## HFS+ images are read-only unless a share opts in {#hfsplus-images-are-writable}

Since v0.44.0 (hfsplus v0.9.0) a share whose image holds HFS+ can be opened for
writing, like FAT, exFAT and ext4: a client can create, change and delete
files, and the times, modes and owners it sets are stored. Before, an HFS+
image was opened read-only.

**Off by default since v0.45.1, and still off.** hfsplus v0.9.0 was not
crash-safe (it held the whole image in memory, rewrote it whole on every
operation and never marked the volume as in use), so v0.45.1 turned writing off
after v0.44.0 and v0.45.0 had written by default (up to 256 MiB). **Since
v0.46.0 (hfsplus v0.10.0)** a writable HFS+ volume is written **in place**:

- **No memory cost.** The image is not copied into memory (an 8 MiB read cache);
  only the sectors a commit changes are written. The 64 GiB ceiling is gone.
- **A crash is visible.** Each commit marks the volume as in use: the unmounted
  bit and the journaled bit are cleared in the first header write and restored
  at the end, so `fsck_hfs -q` sees a crash, journaled volumes included.
- **A failed write stops the volume.** After a failed flush the volume refuses
  writes (`ErrVolumeFailed`) until the share is reopened.

It stays **opt-in**, because of what hfsplus has not finished: a durable small
write costs 3 fsyncs per commit and is about 3x slower than on macOS
(go-filesystems/hfsplus#35), the crash tests are not hardened yet
(go-filesystems/hfsplus#37), and after a crash a journaled volume mounts without
its journal until `diskutil enableJournal`. The default can come back when
those are fixed. Back the image up first. A share opts in with
`hfsplus_writable_max`, which is `"0"` (never) by default (a number of bytes, or
one with K, M, G or T):

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"   # opt in
}
```

Since v0.46.0 the size no longer bounds memory. It stays a size, so that a
configuration written for v0.44.0 to v0.45.1 means what it did (a share that
said `"256M"` does not start writing a 2 TiB image); a large value such as
`"1T"` is the plain opt-in.

With the default, above the size, or when the size is unknown, the share opens
**read-only**, still with range reads and times, and the server writes one line
naming the image and the reason, for instance with no setting:

```text
/srv/mac.img is an HFS+ volume and hfsplus_writable_max is 0, the default: HFS+ images are written only when a share sets a size, because a durable small write is about 3x slower than on macOS (go-filesystems/hfsplus#35), the crash tests are not finished (go-filesystems/hfsplus#37), and after a crash a journaled volume mounts without its journal until diskutil enableJournal, so mac is read-only
```

and above the size:

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes), so mac is read-only
```

**A reload follows the setting (since v0.46.0).** When a reload changes whether
an HFS+ image is written (the setting removed, added, or moved across the image's
size), the image is opened again: removing the opt-in makes the share read-only
at once, adding it makes the share writable. Before v0.46.0 the open driver was
kept, so a share stayed writable after its opt-in was removed until the server
restarted. A size changed from one writable value to another changes nothing.

A **journaled** volume (made by `hdiutil create -fs "Journaled HFS+"` or by
macOS) is written only when its journal is empty, which is what a clean
unmount leaves. A journal that holds transactions, or one that cannot be
verified, would be replayed by macOS over what the server writes, so the driver
refuses it and **the share opens read-only instead**. The server keeps
starting; it writes one line when the share opens, naming the image, the share
and why:

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

To make such a share writable, have the journal replayed first: mount the
image on a Mac and eject it cleanly, or run `fsck_hfs` on it. While a journaled
volume is being written its journaled bit is cleared; after a crash it stays
cleared, so the volume mounts non-journaled until `diskutil enableJournal`. A
share with `read_only = true`, a device and a partition never try the writable
open.

The image is opened a second time by its name for writing (hfsplus takes a
path), and only if that name is still the very file the share holds; otherwise
the share opens read-only and says so.

**Another writer.** A second writer on the same image would corrupt it. A
writable open takes an exclusive advisory `flock` on the image (Linux, macOS and
the BSDs); if another process holds it, for instance a second fileshare on the
same image, the share opens read-only and says so. The lock is advisory: a macOS
mount does not take it, and on other platforms there is no lock.

⛔ **Do not write an image that is mounted**, by this server or by anything
else: neither side knows about the other's changes. Serve an image that nothing
else has open.

## A writable image the OS has mounted is served read-only {#an-image-the-os-has-mounted}

Since v0.45.0, before **any** writable image share (FAT32, exFAT, ext4, UFS,
btrfs, XFS, ZFS, HFS+: whatever the driver) is opened for writing, the server
asks the OS whether it has that image attached or mounted. If it has, the share
opens **read-only**, is marked read-only, and the server writes one line naming
the image, the OS's device and mount, and the share:

```text
/srv/disk.img is in use by the OS (/dev/loop3, mounted at /mnt/disk), and a second writer would corrupt it, so disk is read-only
```

The advisory `flock` above cannot do this: the OS takes no lock when it mounts an
image. The image is matched by device and inode, never by comparing path strings,
so a symbolic link or a hard link to the mounted file is caught too.

| OS | How it is asked |
| --- | --- |
| Linux | `/sys/block/loop*/loop/backing_file` of each loop device (what `mount -o loop`, `losetup` and udisks make); the mount point comes from `/proc/self/mountinfo`. An attached loop device counts even when nothing is mounted on it. |
| macOS | `hdiutil info -plist`, the `image-path` of each attached image, one call per load, bounded by a 15 s timeout. An image attached with `-nomount` counts. |
| FreeBSD, other BSDs, Windows | **Cannot be detected.** The share is writable, as before. |

**When the check itself fails** (no `/sys/block` in a container, `hdiutil`
missing or timing out, output that is not a property list), nobody knows whether
the image is mounted. `image_mount_unknown` decides:

```hcl
share "disk" {
  image               = "/srv/disk.img"
  image_mount_unknown = "read_only"   # the default; "write" serves it writable anyway
}
```

The default is `read_only`: a read-only share is a visible state with a log line
that names its cause, and a person can fix it; a writable share under a mount
corrupts both sides' data, and nobody sees it until later. `write` is for a host
where the check cannot work (a container with no sysfs) and nobody mounts
images. An image the OS is known to have mounted is read-only whatever this says.
A backing path the check cannot read (permission denied, a symbolic-link loop) is
"could not be judged", not "another image": it goes to the same setting. Only a
path that is not there is skipped. On Linux a loop device whose backing file was
deleted is matched through the kernel's own record (`LOOP_GET_STATUS64`) when the
process may open `/dev/loopN`, and is otherwise not matched.

**What the check cannot see**

- **A mount made after the share opened the image.** The check runs when an image
  is opened: at start, for a share created through the admin API, and when a
  reload rebuilds the shares (a change of people or lists) and finds the image
  mounted since, in which case that share gets a read-only open of its own. A
  reload that changes nothing does not look again, and nothing watches for a later
  `hdiutil attach` or `mount -o loop`. Stop the share, or restart the server,
  before mounting an image it serves writable.
- **Mounts that go through no loop device or `hdiutil`:** `qemu-nbd -c` (an nbd
  device), FUSE mounts (`fuse2fs`, `guestmount`), and a virtual machine or other
  program that has the image open.
- Other operating systems (see the table), and a Linux loop device attached in
  another mount namespace.
- A share made read-only because of a mount stays read-only on a reload even if
  the mount has gone; restart it to open it for writing again.

## A partition, for any filesystem

A disk image holding FAT32, ext4 or exFAT has a partition table in front of it
as often as one holding XFS does — and detection reads offset zero, where a
partitioned image has the *table*. So the partition is chosen first, and the
driver is handed a view of it:

```hcl
share "photos" {
  image           = "/srv/disk.img"
  partition_label = "photos"          # what lsblk calls PARTLABEL
}
```

Three ways to name one, and **only one may be given** — two that disagree would
serve whichever the code tried first:

| | |
|---|---|
| `partition = 2` | counting from 1, the way partitioning tools print them |
| `partition_label = "photos"` | the GPT partition name |
| `partition_uuid = "…"` | the GPT unique GUID — Linux's `PARTUUID` |

{{< callout type="error" >}}
**An index moves**

A disk repartitioned, a tool that writes entries in another order, an image
restored with one partition fewer — and `partition = 2` names something
else, **silently**, because a filesystem is still found there. A label or a
UUID names the partition itself, which is why fstabs stopped using indexes
years ago. The index is here for MBR images, which have neither.
{{< /callout >}}

A share that chose a partition is **read-only**, and says so at startup: the
driver's offsets are the partition's while the file underneath is the whole
disk, so a write would land at that offset from the start of the *image* — on
the partition table, as often as not.

## Naming a filesystem turns detection off

{{< callout type="error" >}}
**That is the point, and it is the risk**

The image is opened as *that* or refused. A FAT32 image told it is XFS does
not become an XFS share; it fails to start, saying what it was asked to
open it as.
{{< /callout >}}

`filesystem` is accepted for the sniffable ones too, and then the two are
compared: a share that says `ext4` over a FAT32 image is refused with *the
share says ext4 and the image holds fat32*. That is how a site refuses a
misdetection rather than discovering one later.
[`check`]({{< relref "/configuration/check.md" >}}) marks a named driver with an asterisk, because that row was
not recognised — it was asserted.

## A device, not only an image

```hcl
share "photos" {
  image = "/dev/sda"        # a device node, not a file
}
```

**No privilege is involved.** Nothing here mounts anything — the
[`go-filesystems`](https://github.com/go-filesystems) drivers read ext4, xfs and
the rest in user space — so there is no `mount(2)` and therefore no
`CAP_SYS_ADMIN`. Opening a device is an ordinary `open(2)`, governed by the
permissions on the node:

| | owner | mode | enough |
|---|---|---|---|
| Linux `/dev/sda` | `root:disk` | 0660 | membership of `disk` |
| macOS `/dev/disk0` | `root:operator` | 0640 | membership of `operator`, plus Full Disk Access |

A refusal says which group, rather than `permission denied` — the answer is
never `sudo`.

⛔ **A device share is read-only**, for the same reason as a share that chose a
partition: writing to a live disk is not a decision to make on your behalf.

{{< callout type="error" >}}
**The device is opened exclusively, and that is about correctness, not caution**

If the kernel has a filesystem mounted from it, the filesystem's page cache
holds newer metadata than the device does — so a raw reader sees a
directory block from before an update beside an inode block from after it,
a state that never existed on disk at any one moment. `O_EXCL` on a block
device is the kernel's own primitive for "nobody else, a mount included";
it is what `mkfs` and `fsck` use. A device in use is refused, by name, with
the reason.
{{< /callout >}}

Two things were measured rather than assumed:

- `Stat().Size()` is **0** for a device, so the length is asked of the device
  itself. Seeking to the end answers on Linux and returns **0 with no error**
  on macOS, which is why Darwin uses `DKIOCGETBLOCKCOUNT` instead.
- A **raw** node (`/dev/rdiskN`, or `O_DIRECT` on Linux) refuses an unaligned
  read, and reading a two-byte field at offset 11 is what parsing a FAT BPB
  *is*. Reads to a device are rounded out to whole blocks; without that,
  `/dev/rdisk4` reported `unknown filesystem`.

## A directory, not only an image

Since v0.8.0 a share may serve a directory of the host — a container's volume,
say — instead of an image:

```hcl
share "photos" {
  directory = "/data/photos"
  allow     = ["@family"]
}
```

It is [go-filesystems/osfs](https://github.com/go-filesystems/osfs), which
reaches the tree through an **`os.Root`**: `..` that climbs out, an absolute
path, and a symbolic link leading out are refused by the kernel-backed root,
not by a string test. A client may *create* a link to `/etc`; nothing will
follow it. A FIFO planted in the tree is refused rather than left to hang the
server.

{{< callout type="error" >}}
**One share may not hold another (since v0.17.0)**

A directory share whose tree holds another share's image or directory is
**refused at start**, and by the admin API: whoever may use the outer
share would read and write the inner one without being allowed it —
anonymous NFS included. So are two shares on the same source, except two
different partitions of one disk image.
{{< /callout >}}

{{< callout type="info" >}}
**Not behind the image lock**

An image driver owns one file and promises nothing about two calls at once,
which is why [every image is wrapped in one lock]({{< relref "/operations/locking.md" >}}).
A host tree is the kernel's, which serialises what must be serialised per
call and no more; one mutex in front of every file of every client would be
the slowest thing a file server can do.
{{< /callout >}}

⛔ A share has an `image` **or** a `directory`, never both, and never neither.
`filesystem` and `partition` are for an image: a directory share naming one is
refused, because saying one means the share was meant to be an image.
`read_only = true` works on a directory as on an image; a share that says it
and also lists `writers` is refused, because `read_only` would win.

The capacity a client is told is that of the filesystem the tree lives on; a
platform that cannot say leaves it at zero, which the protocols read as
"unknown" rather than "full".

## What SMB does with a time it cannot store {#what-smb-does-with-a-time-it-cannot-store}

A time a client sets that the storage cannot keep is **ignored** by default.
Since v0.42.0 a share can choose, with `unstorable_times`. A time is unstorable
when it is a CreationTime that differs from the reported one, a ChangeTime sent
alone, or an access or write time on an image whose driver cannot set times
(NTFS, APFS, ISO 9660, SquashFS; a host directory and a FAT, exFAT, ext4 or HFS+ image store those):

```hcl
share "photos" {
  directory        = "/data/photos"
  unstorable_times = "log"      # "ignore" (the default), "refuse" or "log"
}
```

`refuse` answers the client with an error, and `log` writes one line naming the
share and the path. The exact cases, per kind of share, are in
[Real file times]({{< relref "/protocols/times.md#when-unstorable" >}}).
