# Shares, filesystems and partitions

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
