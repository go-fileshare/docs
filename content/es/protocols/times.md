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
establecer fechas.

### Elegir qué ocurre: `unstorable_times` {#unstorable-times}

Desde la v0.42.0, un recurso compartido indica qué ocurre con una fecha así
mediante `unstorable_times` (la opción `UnstorableTimes` de `go-filesystems/smb`):

```hcl
share "attic" {
  directory        = "/srv/attic"
  unstorable_times = "refuse"   # "ignore" (por defecto), "refuse" o "log"
}
```

| Valor | Lo que oye el cliente | Lo que se conserva |
|---|---|---|
| `ignore` (también si falta el ajuste) | éxito | el campo se descarta, como hace Samba |
| `refuse` | `STATUS_NOT_SUPPORTED`, para que una herramienta de copia de seguridad nunca crea que se conservó una fecha | no se aplica nada de esa solicitud |
| `log` | éxito | el campo se descarta, y una línea por solicitud nombra el recurso, la ruta y los campos, escrita donde el servidor escribe sus propias líneas |

Cualquier otro valor se rechaza al leer la configuración, con un error que
nombra el recurso y las tres palabras. El ajuste pertenece a SMB: se rechaza un
recurso cuyo `protocols` excluye SMB si lo establece. No hay un valor por
defecto para todo el servidor; se indica en cada recurso. Un recurso creado
mediante la API de administración usa `ignore`.

### Cuándo no se puede almacenar una fecha {#when-unstorable}

Una solicitud se juzga campo por campo, y basta un campo no almacenable:
`refuse` rechaza toda la solicitud y `log` escribe su línea. Una fecha no es
almacenable cuando:

1. **CreationTime** se establece en un valor distinto del que informa el
   controlador (o el controlador no informa ninguno). No hay forma de establecer
   una fecha de creación. Un valor igual al informado se acepta: no hay nada que
   cambiar.
2. **ChangeTime** se envía **sola**, con un valor distinto del informado.
   Enviada junto con una fecha de acceso o de escritura se acepta, si el
   controlador puede establecer fechas, porque al establecerlas se actualiza la
   fecha de cambio a «ahora».
3. **Una fecha de acceso o de escritura**, en un recurso cuyo controlador no
   puede establecer fechas (`Chtimes`): cualquier valor real, igual al actual o
   no.

Tres cosas no las decide la política. Una fecha de 0 (conservar), -1 (congelar)
o -2 (reanudar) nunca es no almacenable. -1 en un controlador que puede
establecer fechas pero no informar de ellas es siempre `STATUS_NOT_SUPPORTED`,
en todos los modos: es una capacidad que falta, no un campo descartado (afecta a
las imágenes ufs, btrfs, xfs y zfs). Y una solicitud sobre un recurso de **solo
lectura** (`read_only = true`, un dispositivo, un recurso que eligió una
partición, una persona sin permiso de escritura) se responde con
`STATUS_MEDIA_WRITE_PROTECTED` antes de todo esto, en todos los modos.

Lo que un recurso puede almacenar depende de su controlador, en las versiones
que fija esta versión:

| Recurso | CreationTime | ChangeTime sola | Fecha de acceso o de escritura |
|---|---|---|---|
| **Directorio** (host) | no almacenable salvo que sea igual a la fecha de creación informada | no almacenable salvo que sea igual a la informada | almacenada (una ChangeTime enviada con ella se acepta) |
| **Imagen, controlador con `Chtimes`**: ext4, ufs, btrfs, xfs, zfs | no almacenable salvo que sea igual a la informada (solo ext4 informa una: en los demás, siempre) | igual | almacenada (una ChangeTime enviada con ella se acepta) |
| **Imagen, controlador sin `Chtimes`**: FAT, exFAT, NTFS, HFS+, APFS, ISO 9660, SquashFS | igual (FAT, exFAT, NTFS y HFS+ informan una; los demás nunca) | igual, **también** cuando se envía con una fecha de acceso o de escritura | **siempre no almacenable** |

Los controladores con `Chtimes` son osfs (`osfs.go:383`), ext4 (`metadata.go:93`),
ufs (`metadata.go:63`), btrfs (`fs.go:382`), xfs (`fs.go:521`) y zfs
(`metadata.go:94`); los demás no tienen. Antes de la v0.42.0 el bloqueo que
fileshare coloca delante de cada controlador de imagen ocultaba `Chtimes`, de
modo que toda imagen descartaba las fechas de acceso y de escritura de un
cliente, ext4 incluida; desde la v0.42.0 lo transmite exactamente cuando el
controlador lo tiene. Las imágenes NTFS las lee un controlador de solo lectura,
así que no se almacena nada diga lo que diga la política; esta solo decide lo que
se le comunica al cliente.

Qué clientes provocan cada caso: el Explorador de Windows y `robocopy /COPY:T`
envían una CreationTime, que un recurso de cualquier tipo no puede conservar si
cambia; `touch -d` y `cp -p` de Linux envían solo fechas de acceso y de
escritura (el cliente del kernel añade una ChangeTime de «ahora»), que un
directorio y un controlador con `Chtimes` almacenan, y uno sin él no.
