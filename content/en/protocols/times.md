---
title: "Real file times"
linkTitle: "File times"
weight: 60
description: "Which shares report each file's own times over NFS, SMB, SFTP and WebDAV, what each protocol says when a time is not recorded, and what is not supported yet."
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

## Not supported yet

A client that **sets** a time over SMB (`SET_INFO`) is not obeyed: the time
stays what the filesystem recorded (go-filesystems/smb#45).
