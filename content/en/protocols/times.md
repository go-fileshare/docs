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

### When a time cannot be stored {#when-unstorable}

A request is judged field by field, and one unstorable field is enough: `refuse`
refuses the whole request, `log` writes its line. A time is unstorable when:

1. **CreationTime** is set to a value different from the one the driver reports
   (or the driver reports none). There is no way to set a creation time. A value
   equal to the reported one is accepted: there is nothing to change.
2. **ChangeTime** is sent **alone**, with a value different from the reported
   one. Sent together with an access or write time it is accepted, when the
   driver can set times, because setting them refreshes the change time to now.
3. **An access or write time**, on a share whose driver cannot set times
   (`Chtimes`): any real value, equal to the current one or not.

Three things are not decided by the policy. A time of 0 (keep), -1 (freeze) or
-2 (resume) is never unstorable. -1 on a driver that can set times but cannot
report them is always `STATUS_NOT_SUPPORTED`, in every mode: that is a missing
capability, not a dropped field (no share fileshare serves is of that kind). And
a request on a **read-only** share (`read_only = true`, a device, a share that
chose a partition, a person who may not write) is answered
`STATUS_MEDIA_WRITE_PROTECTED` before any of this, in every mode.

| Share | CreationTime | ChangeTime alone | Access or write time |
|---|---|---|---|
| **Directory** (host) | unstorable unless equal to the reported birth time | unstorable unless equal to the reported one | stored (a ChangeTime sent with it is accepted) |
| **Image**: FAT, exFAT, NTFS, ext4, HFS+ | unstorable unless equal to the reported one | unstorable unless equal, **also** when sent with an access or write time | **always unstorable** |
| **Other image types** | always unstorable (nothing is reported to compare with) | always unstorable | **always unstorable** |

An image's access and write times are unstorable because the lock fileshare puts
in front of every image driver does not carry `Chtimes`, even for ext4, whose
driver has one. NTFS images are read by a read-only driver, so nothing is stored
whatever the policy says; the policy decides only what the client is told.

Which clients hit which case: Windows Explorer and `robocopy /COPY:T` send a
CreationTime, which a share of any kind cannot keep unless it is unchanged; Linux
`touch -d` and `cp -p` send only access and write times (the kernel client adds a
ChangeTime of "now"), which a directory stores and an image cannot.
