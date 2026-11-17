---
title: "automount"
weight: 10
description: "Mount an SMB share in the operating system's file manager without a password dialog, and unmount it."
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.2
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
- **Linux (Ubuntu 24.04, gvfs 1.54.4): measured** in a headless session
  (`dbus-run-session`, gvfsd, gvfsd-smb and gvfsd-fuse): the same sequence as on
  macOS, a wrong password reaching `gvfs.ErrAuthFailed`. Not measured: a real GNOME
  desktop session. The run found a defect in go-freedesktop/gvfs v0.1.0 (`/`
  returned as the path when gvfs-fuse is not serving the session), fixed in
  v0.1.1; then `Path()` is the `smb://` URI, which GIO can open and `os` calls
  cannot.
- **Windows 11 25H2 (arm64, QEMU): measured as a standard user and as an
  administrator** (v0.1.2 adds the standard-user run): a wrong password reaching
  `*win32.SMBPortError`, map `Z:`, read and write through it, unmount, `Z:` gone.
  Not measured: the letter's appearance in Explorer's "This PC" (an ssh logon has
  its own logon session, so the check is through `os` calls).

It is used by [theca](https://github.com/go-theca/theca) `serve --mount`.
