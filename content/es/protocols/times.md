---
title: "Las fechas reales de los archivos"
linkTitle: "Fechas de archivos"
weight: 60
description: "Qué recursos compartidos informan las fechas propias de cada archivo por NFS, SMB, SFTP y WebDAV, qué dice cada protocolo cuando una fecha no está registrada y qué aún no se admite."
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
| Imágenes exFAT, NTFS, ext4, HFS+ | todavía no (exfat#29, ntfs#32, ext4#41, hfsplus#25): se responde como antes |
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

## Aún no se admite {#not-supported-yet}

Un cliente que **establece** una fecha por SMB (`SET_INFO`) no es obedecido: la
fecha sigue siendo la que registró el sistema de archivos
(go-filesystems/smb#45).
