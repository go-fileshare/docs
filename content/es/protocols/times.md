---
title: "Las fechas reales de los archivos"
linkTitle: "Fechas de archivos"
weight: 60
description: "Qué recursos compartidos informan las fechas propias de cada archivo por NFS, SMB, SFTP y WebDAV, qué dice cada protocolo cuando una fecha no está registrada y qué ocurre cuando un cliente establece una fecha por SMB."
tags: [protocolos, fechas]
---

Desde la v0.39.0, las fechas de un archivo son las que su sistema de archivos
registró, no la hora de la petición. Esto requiere la capacidad opcional
`TimesReader` de `go-filesystems/interface` v0.6.0, que un controlador tiene o
no tiene.

## Qué recursos informan fechas reales {#which-shares-report-real-times}

| Recurso | Fechas reales |
|---|---|
| **Directorio** (anfitrión) | sí: acceso, modificación, cambio y, si el anfitrión la guarda, creación, al nanosegundo |
| **Imagen FAT** | sí, con la resolución de FAT: modificación 2 s, acceso un día, creación 10 ms; no hay fecha de cambio |
| **Imagen exFAT** (desde v0.40.0) | sí: creación y modificación 10 ms, acceso 2 s; no hay fecha de cambio |
| **Imagen NTFS** (desde v0.40.0) | sí: creación, modificación, cambio y acceso, 100 ns |
| **Imagen ext4** (desde v0.40.0) | sí: acceso, modificación y cambio a 1 ns (1 s en los inodos ext2/ext3 de 128 bytes); creación solo si el inodo es lo bastante grande |
| **Imagen HFS+** (desde v0.40.0) | sí: creación, modificación, cambio y acceso, 1 s |
| Otros tipos de imagen | se responde como antes |
| Recurso servido como la persona (worker `remote`) | sí, si el directorio del worker es un directorio del anfitrión: la fecha cruza la frontera del worker |

Una fecha se compara con la resolución **más gruesa** entre la del formato y
la del protocolo: SFTP transporta segundos enteros, el `getlastmodified` de
WebDAV también, SMB 100 ns, NFS 1 ns.

## Cuando una fecha no está registrada {#when-a-time-is-not-recorded}

| Protocolo | Qué se envía |
|---|---|
| **SMB** | 0, «no especificado». Un controlador **sin** la capacidad sigue respondiendo la hora actual. |
| **NFS** | Una fecha de acceso o de cambio ausente pasa a ser la de modificación; una fecha de modificación ausente pasa a ser la hora de arranque del servidor. |
| **SFTP** | Una fecha de modificación ausente borra el indicador `ACMODTIME`: el cliente no ve ninguna fecha en lugar de una falsa. |
| **WebDAV** | `creationdate` es la fecha de creación; una fecha no registrada no aparece en la respuesta. |

## Fechas que un cliente establece por SMB {#setting-times}

Desde la v0.41.0 (`go-filesystems/smb` v0.10.0,
go-filesystems/smb#45), las fechas de acceso y de modificación que un cliente
establece por SMB (`SET_INFO`, como hacen `touch`, `robocopy /COPY:T` y una
copia en el Explorador) se aplican al archivo, en un recurso compartido de
**directorio**. Una fecha de 0 la deja como está; -1 la congela y -2 la
reanuda, por identificador abierto.

Una fecha que no se puede almacenar se **ignora por defecto**: la solicitud
tiene éxito y ese campo se descarta, como hace Samba, para que las copias que
conservan las fechas no fallen. Esto abarca la fecha de creación, una fecha de
cambio enviada sola y cualquier fecha en un controlador que no puede
establecer fechas. fileshare no ofrece un ajuste para esto: todos los recursos
compartidos usan el valor por defecto.
