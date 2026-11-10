---
title: "Real file times"
linkTitle: "File times"
weight: 60
description: "Which shares report each file's own times over NFS, SMB, SFTP and WebDAV, what each protocol says when a time is not recorded, and what happens when a client sets a time over SMB."
tags: [protocols, timestamps]
---

Since v0.39.0 a file's times are the ones its filesystem recorded, not the time
of the request. This needs the optional `TimesReader` capability of
`go-filesystems/interface` v0.6.0, which a driver either has or does not.

## Which shares report real times

| Share | Real times |
|---|---|
| **Directory** (host) | yes: access, modification, change and, where the host keeps it, birth time, to the nanosecond |
| **FAT image** | yes, at FAT's resolution: modification 2 s, access one day, creation 10 ms; there is no change time |
| **exFAT image** (since v0.40.0) | yes: creation and modification 10 ms, access 2 s; there is no change time |
| **NTFS image** (since v0.40.0) | yes: creation, modification, change and access, 100 ns |
| **ext4 image** (since v0.40.0) | yes: access, modification and change to 1 ns (1 s on 128-byte ext2/ext3 inodes); creation only where the inode is large enough |
| **HFS+ image** (since v0.40.0) | yes: creation, modification, change and access, 1 s |
| Other image types | answered as before |
| Share served as its people (`remote` worker) | yes, when the worker's directory is a host directory: the time crosses the worker boundary |

A time is compared at the **coarser** of the format's resolution and the
protocol's own: SFTP carries whole seconds, a WebDAV `getlastmodified` whole
seconds, SMB 100 ns, NFS 1 ns.

## When a time is not recorded

| Protocol | What is sent |
|---|---|
| **SMB** | 0, "not specified". A driver **without** the capability still answers the current time. |
| **NFS** | An access or change time that is missing becomes the modification time; a missing modification time becomes the time the server started. |
| **SFTP** | A missing modification time clears the `ACMODTIME` flag: the client sees no time rather than a wrong one. |
| **WebDAV** | `creationdate` is the birth time; a time that is not recorded is absent from the answer. |

## Times a client sets over SMB {#setting-times}

Since v0.41.0 (`go-filesystems/smb` v0.10.0,
go-filesystems/smb#45), the access and write times a client sets over SMB
(`SET_INFO`, as `touch`, `robocopy /COPY:T` and a copy in Explorer do) are
applied to the file, on a **directory** share. A time of 0 leaves that time as
it is; -1 freezes it and -2 resumes it, per open handle.

A time that cannot be stored is **ignored by default**: the request succeeds and
that field is dropped, as Samba does, so that copies which preserve times do not
fail. This covers the creation time, a change time sent alone, and any time on a
driver that cannot set times.

### Choosing what happens: `unstorable_times` {#unstorable-times}

Since v0.42.0 a share says what happens to such a time, with `unstorable_times`
(the `UnstorableTimes` option of `go-filesystems/smb`):

```hcl
share "attic" {
  directory        = "/srv/attic"
  unstorable_times = "refuse"   # "ignore" (the default), "refuse" or "log"
}
```

| Value | What the client hears | What is kept |
|---|---|---|
| `ignore` (also when the setting is absent) | success | the field is dropped, as Samba does |
| `refuse` | `STATUS_NOT_SUPPORTED`, so a backup tool never believes a time was kept | nothing of that request is applied |
| `log` | success | the field is dropped, and one line per request names the share, the path and the fields, written where the server writes its own lines |

Any other value is refused when the configuration is read, with an error that
names the share and the three words. The setting belongs to SMB: a share whose
`protocols` leave SMB out is refused if it sets it. There is no server-wide
default; say it on each share. A share created through the admin API uses
`ignore`.
