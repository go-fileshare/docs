---
title: "automount"
weight: 10
description: "Eine SMB-Freigabe ohne Passwortdialog im Dateimanager des Betriebssystems einhängen und wieder aushängen."
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.0
hängt eine SMB-Freigabe **ohne jeden Passwortdialog** im Dateimanager des
Betriebssystems ein und wieder aus. Es ist eine kleine Fassade in reinem Go
(ohne cgo) über drei vorhandenen Backends. Diese Seite ist aus seiner
[README](https://github.com/go-fileshare/automount/blob/main/README.md)
geschrieben; wo sie vom Repository abweicht, gilt das Repository.

| System | Backend | Was erscheint |
|---|---|---|
| macOS | [go-macos/netfs](https://github.com/go-macos/netfs) (NetFS, `NoUI`) | ein Volume unter `/Volumes`, in der Finder-Seitenleiste |
| Linux (GNOME) | [go-freedesktop/gvfs](https://github.com/go-freedesktop/gvfs) (beantwortet GVfs' eigenes `AskPassword`) | ein gvfs-Ort, in Dateien |
| Windows | [go-mswin/win32](https://github.com/go-mswin/win32) (`MSFT_SmbMapping.Create`, jeder TCP-Port, Standardbenutzer) | ein Laufwerksbuchstabe, in „Dieser PC“ |

Jedes andere GOOS lässt sich kompilieren und antwortet `automount.ErrUnsupported`.

```go
u, _ := url.Parse("smb://127.0.0.1:4455/theca")
m, err := automount.Mount(ctx, automount.Share{URL: u, User: "alice", Password: pw}, automount.Options{})
if err != nil { /* errors.Is(err, netfs.ErrAuth) etc. */ }
fmt.Println(m.Path())   // /Volumes/theca, Z:\, /run/user/1000/gvfs/...
defer m.Unmount(ctx)
```

`Mount` lehnt vor dem Zugriff auf das Betriebssystem ab: eine leere oder
nicht-`smb`-URL, einen fehlenden Host oder eine fehlende Freigabe, eine Freigabe
mit Unterpfad, einen ungültigen Port, Benutzerangaben in der URL, ein ungültiges
oder belegtes Windows-Laufwerk und `ReadOnly`, wo es sich nicht einhalten lässt
(nur macOS). Das Passwort gelangt nie in einen Argumentvektor, die Umgebung, eine
URL, eine Fehlermeldung oder ein Protokoll.

## Gemessen, und wo {#measured}

- **macOS 27.2.1 (arm64): gemessen.** Der SMB-Server von
  [go-filesystems/smb](https://github.com/go-filesystems/smb) läuft im
  Testprozess auf `127.0.0.1:<zufälliger Port>`; die Fassade hängt ihn ein, eine
  Datei wird über `Path()` gelesen und geschrieben, `Unmount` entfernt das Volume.
  Ein falsches Passwort scheitert mit `netfs.ErrAuth`.
- **Linux: nicht live gemessen** (die Unit-Tests ersetzen die GVfs-Aufrufe).
- **Windows: nicht live gemessen** (die Unit-Tests ersetzen die win32-Aufrufe).
  Der `MapSMB`-Aufruf als Standardbenutzer ist in go-mswin/win32 gemessen; eine
  Einbindung aus einer interaktiven Desktop-Sitzung nicht.

Es wird von [theca](https://github.com/go-theca/theca) `serve --mount` verwendet.
