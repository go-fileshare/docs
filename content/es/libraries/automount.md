---
title: "automount"
weight: 10
description: "Montar un recurso SMB en el gestor de archivos del sistema operativo sin cuadro de contraseña, y desmontarlo."
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.2
monta un recurso SMB en el gestor de archivos del sistema operativo **sin ningún
cuadro de contraseña**, y lo desmonta. Es una pequeña fachada en Go puro (sin
cgo) sobre tres motores existentes. Esta página está escrita a partir de su
[README](https://github.com/go-fileshare/automount/blob/main/README.md); donde
difiere del repositorio, manda el repositorio.

| Sistema | Motor | Lo que aparece |
|---|---|---|
| macOS | [go-macos/netfs](https://github.com/go-macos/netfs) (NetFS, `NoUI`) | un volumen bajo `/Volumes`, en la barra lateral del Finder |
| Linux (GNOME) | [go-freedesktop/gvfs](https://github.com/go-freedesktop/gvfs) (responde al `AskPassword` propio de GVfs) | una ubicación gvfs, en Archivos |
| Windows | [go-mswin/win32](https://github.com/go-mswin/win32) (`MSFT_SmbMapping.Create`, cualquier puerto TCP, usuario estándar) | una letra de unidad, en «Este equipo» |

Cualquier otro GOOS compila y responde `automount.ErrUnsupported`.

```go
u, _ := url.Parse("smb://127.0.0.1:4455/theca")
m, err := automount.Mount(ctx, automount.Share{URL: u, User: "alice", Password: pw}, automount.Options{})
if err != nil { /* errors.Is(err, netfs.ErrAuth) etc. */ }
fmt.Println(m.Path())   // /Volumes/theca, Z:\, /run/user/1000/gvfs/...
defer m.Unmount(ctx)
```

`Mount` rechaza, antes de tocar el sistema operativo, una URL nula o no `smb`,
un host o recurso ausente, un recurso con subruta, un puerto incorrecto,
información de usuario en la URL, una unidad de Windows inválida o en uso, y
`ReadOnly` donde no puede cumplirse (solo macOS). La contraseña nunca pasa a un
vector de argumentos, al entorno, a una URL, a un mensaje de error ni a un
registro.

## Medido, y dónde {#measured}

- **macOS 27.2.1 (arm64): medido.** El servidor SMB de
  [go-filesystems/smb](https://github.com/go-filesystems/smb) corre en el
  proceso de prueba en `127.0.0.1:<puerto aleatorio>`; la fachada lo monta, se
  lee y escribe un archivo a través de `Path()`, `Unmount` quita el volumen. Una
  contraseña incorrecta falla con `netfs.ErrAuth`.
- **Linux (Ubuntu 24.04, gvfs 1.54.4): medido** en una sesión sin pantalla
  (`dbus-run-session`, gvfsd, gvfsd-smb y gvfsd-fuse): la misma secuencia que en
  macOS, y una contraseña incorrecta llega a `gvfs.ErrAuthFailed`. No medido: una
  sesión real de escritorio GNOME. La prueba encontró un defecto en
  go-freedesktop/gvfs v0.1.0 (`/` como ruta cuando gvfs-fuse no sirve la sesión),
  corregido en v0.1.1; entonces `Path()` es la URI `smb://`, que GIO puede abrir y
  las llamadas de `os` no.
- **Windows 11 25H2 (arm64, QEMU): medido como usuario estándar y como
  administrador** (v0.1.2 añade la ejecución como usuario estándar): una
  contraseña incorrecta llega a `*win32.SMBPortError`, se asigna `Z:`, se lee y
  escribe a través de ella, se desmonta y `Z:` desaparece. No medido: la aparición
  de la letra en «Este equipo» del Explorador (un inicio de sesión ssh tiene su
  propia sesión, así que la comprobación usa llamadas de `os`).

Lo usa [theca](https://github.com/go-theca/theca) `serve --mount`.
