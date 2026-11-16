---
title: "automount"
weight: 10
description: "Mount an SMB share in the operating system's file manager without a password dialog, and unmount it."
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.0
mounts an SMB share in the operating system's file manager **without any
password dialog**, and unmounts it. It is a small pure-Go facade (no cgo) over
three existing backends. This page is written from its
[README](https://github.com/go-fileshare/automount/blob/main/README.md); where
it differs from the repository, the repository wins.

| OS | Backend | What appears |
|---|---|---|
| macOS | [go-macos/netfs](https://github.com/go-macos/netfs) (NetFS, `NoUI`) | a volume under `/Volumes`, in the Finder sidebar |
| Linux (GNOME) | [go-freedesktop/gvfs](https://github.com/go-freedesktop/gvfs) (answers GVfs's own `AskPassword`) | a gvfs location, in Files |
| Windows | [go-mswin/win32](https://github.com/go-mswin/win32) (`MSFT_SmbMapping.Create`, any TCP port, standard user) | a drive letter, in "This PC" |

Any other GOOS compiles and answers `automount.ErrUnsupported`.

```go
u, _ := url.Parse("smb://127.0.0.1:4455/theca")
m, err := automount.Mount(ctx, automount.Share{URL: u, User: "alice", Password: pw}, automount.Options{})
if err != nil { /* errors.Is(err, netfs.ErrAuth) etc. */ }
fmt.Println(m.Path())   // /Volumes/theca, Z:\, /run/user/1000/gvfs/...
defer m.Unmount(ctx)
```

`Mount` refuses, before touching the OS, a nil or non-`smb` URL, a missing host
or share, a share with a sub-path, a bad port, user information in the URL, a bad
or used Windows drive, and `ReadOnly` where it cannot be honoured (macOS only).
The password never goes into an argument vector, the environment, a URL, an
error message or a log.

## Measured, and where {#measured}

- **macOS 27.2.1 (arm64): measured.** The SMB server of
  [go-filesystems/smb](https://github.com/go-filesystems/smb) runs in the test
  process on `127.0.0.1:<random port>`; the facade mounts it, a file is read and
  written through `Path()`, `Unmount` removes the volume. A wrong password fails
  with `netfs.ErrAuth`.
- **Linux: not measured** live (the unit tests replace the GVfs calls).
- **Windows: not measured** live (the unit tests replace the win32 calls). The
  `MapSMB` call as a standard user is measured in go-mswin/win32; a mapping made
  from an interactive desktop session was not.

It is used by [theca](https://github.com/go-theca/theca) `serve --mount`.
