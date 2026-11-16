---
title: "automount"
weight: 10
description: "Montar un recurso SMB en el gestor de archivos del sistema operativo sin cuadro de contraseña, y desmontarlo."
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.0
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
- **Linux: no medido** en vivo (las pruebas unitarias reemplazan las llamadas a
  GVfs).
- **Windows: no medido** en vivo (las pruebas unitarias reemplazan las llamadas
  a win32). La llamada `MapSMB` como usuario estándar está medida en
  go-mswin/win32; un montaje hecho desde una sesión de escritorio interactiva,
  no.

Lo usa [theca](https://github.com/go-theca/theca) `serve --mount`.
