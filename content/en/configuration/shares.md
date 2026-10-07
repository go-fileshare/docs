# Shares: images, devices, directories

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
  partition  = 2       # or leave it out: -1, the first data partition
}
```

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

!!! danger "An index moves"
    A disk repartitioned, a tool that writes entries in another order, an image
    restored with one partition fewer — and `partition = 2` names something
    else, **silently**, because a filesystem is still found there. A label or a
    UUID names the partition itself, which is why fstabs stopped using indexes
    years ago. The index is here for MBR images, which have neither.

A share that chose a partition is **read-only**, and says so at startup: the
driver's offsets are the partition's while the file underneath is the whole
disk, so a write would land at that offset from the start of the *image* — on
the partition table, as often as not.

## Naming a filesystem turns detection off

!!! danger "That is the point, and it is the risk"
    The image is opened as *that* or refused. A FAT32 image told it is XFS does
    not become an XFS share; it fails to start, saying what it was asked to
    open it as.

`filesystem` is accepted for the sniffable ones too, and then the two are
compared: a share that says `ext4` over a FAT32 image is refused with *the
share says ext4 and the image holds fat32*. That is how a site refuses a
misdetection rather than discovering one later.
[`check`](check.md) marks a named driver with an asterisk, because that row was
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

!!! danger "The device is opened exclusively, and that is about correctness, not caution"
    If the kernel has a filesystem mounted from it, the filesystem's page cache
    holds newer metadata than the device does — so a raw reader sees a
    directory block from before an update beside an inode block from after it,
    a state that never existed on disk at any one moment. `O_EXCL` on a block
    device is the kernel's own primitive for "nobody else, a mount included";
    it is what `mkfs` and `fsck` use. A device in use is refused, by name, with
    the reason.

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

!!! danger "One share may not hold another (since v0.17.0)"
    A directory share whose tree holds another share's image or directory is
    **refused at start**, and by the admin API: whoever may use the outer
    share would read and write the inner one without being allowed it —
    anonymous NFS included. So are two shares on the same source, except two
    different partitions of one disk image.

!!! note "Not behind the image lock"
    An image driver owns one file and promises nothing about two calls at once,
    which is why [every image is wrapped in one lock](../operations/locking.md).
    A host tree is the kernel's, which serialises what must be serialised per
    call and no more; one mutex in front of every file of every client would be
    the slowest thing a file server can do.

⛔ A share has an `image` **or** a `directory`, never both, and never neither.
`filesystem` and `partition` are for an image: a directory share naming one is
refused, because saying one means the share was meant to be an image.
`read_only = true` works on a directory as on an image.

The capacity a client is told is that of the filesystem the tree lives on; a
platform that cannot say leaves it at zero, which the protocols read as
"unknown" rather than "full".
