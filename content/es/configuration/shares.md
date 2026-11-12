---
title: "Recursos compartidos: imágenes, dispositivos, directorios"
weight: 10
description: "Lo que puede servir un recurso compartido (una imagen de disco, un dispositivo o un directorio del host) y cómo se eligen su sistema de archivos y su partición."
tags: [configuración, recursos compartidos]
---

Un recurso compartido sirve una de tres cosas: una **imagen de disco**, un nodo de
**dispositivo**, o un **directorio del host**. Una imagen y un dispositivo contienen
un sistema de archivos que este programa lee por sí mismo; un directorio es del
propio host.

## Una imagen, y el sistema de archivos que contiene {#an-image-and-the-filesystem-inside-it}

El sistema de archivos de una imagen **se deduce en lugar de declararse**.
[`go-filesystems/detect`](https://github.com/go-filesystems/detect) lee el número
mágico y devuelve el controlador que le corresponde: **fat32, exfat, ext4, ntfs, ufs,
iso9660, squashfs o hfsplus**, todos los controladores de una misma forma,
`OpenReader(io.ReaderAt, int64)`, un sistema de archivos en el desplazamiento cero.

## Los cuatro que no se pueden detectar {#the-four-that-cannot-be-sniffed}

**apfs, btrfs, xfs y zfs** abren cada uno una imagen de *disco* y eligen una
**partición**, así que no hay número mágico que encontrar en el desplazamiento cero.
El recurso compartido indica cuál:

```hcl
share "photos" {
  image      = "/srv/disk.img"
  filesystem = "xfs"
  partition  = 2       # or leave it out: the driver takes the first data partition
}
```

## Las imágenes HFS+ son escribibles {#hfsplus-images-are-writable}

Desde la v0.44.0 (hfsplus v0.9.0) un recurso cuya imagen contiene HFS+ se abre
para escritura, como FAT, exFAT y ext4: un cliente puede crear, modificar y
borrar archivos, y se almacenan las fechas, los modos y los propietarios que
establece. Antes, una imagen HFS+ se abría en solo lectura. La imagen se lee en
memoria al abrir el recurso y cada cambio se escribe de vuelta en el archivo.

**Lo que cuesta.** hfsplus v0.9.0 mantiene toda la imagen en memoria y reescribe
toda la imagen en cada operación (crear, escribir, renombrar, `Chtimes`), de modo
que la memoria de un recurso es el tamaño de su imagen, y también lo es la E/S de
cada cambio (go-filesystems/hfsplus#33 es la solución de fondo). Por eso un
recurso abre una imagen HFS+ para escritura solo hasta `hfsplus_writable_max`,
256 MiB por defecto (`"0"` nunca; un número de bytes, o uno con K, M, G o T):

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"
}
```

Por encima, o si se desconoce el tamaño, el recurso se abre en **solo lectura**,
aún con lectura por rangos y fechas, y el servidor escribe una línea que nombra
la imagen, el tamaño y el límite:

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes): a writable HFS+ image is held in memory and rewritten whole on every change, so mac is read-only
```

Un volumen **con diario** (creado por `hdiutil create -fs "Journaled HFS+"` o por
macOS) solo se escribe cuando su diario está vacío, que es lo que deja un
desmontaje limpio. Un diario con transacciones, o uno que no se puede
verificar, sería reproducido por macOS sobre lo que escribe el servidor, así que
el controlador lo rechaza y **el recurso se abre en solo lectura**. El servidor
sigue arrancando; al abrir el recurso escribe una línea que nombra la imagen, el
recurso y el motivo:

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

Para que un recurso así sea escribible, hay que reproducir primero el diario:
monte la imagen en un Mac y expúlsela limpiamente, o ejecute `fsck_hfs` sobre
ella. (El controlador no tiene una comprobación del diario que funcione sin la
imagen en memoria, así que este caso lee la imagen, hasta el límite, antes de
volver atrás.) Un recurso con `read_only = true`, un dispositivo y una partición
nunca intentan la apertura para escritura.

**Otro escritor.** La imagen se vuelca entera, así que un segundo escritor sería
sobrescrito por el siguiente volcado. La apertura para escritura toma un bloqueo
consultivo exclusivo `flock` sobre la imagen (Linux, macOS y los BSD); si otro
proceso lo tiene, por ejemplo un segundo fileshare sobre la misma imagen, el
recurso se abre en solo lectura y lo dice. El bloqueo es consultivo: un montaje
de macOS no lo toma, y en otras plataformas no hay bloqueo.

⛔ **No escriba una imagen que esté montada**, ni con este servidor ni con otra
cosa: ninguna de las partes conoce los cambios de la otra. Sirva una imagen que
nada más tenga abierta.

## Una partición, para cualquier sistema de archivos {#a-partition-for-any-filesystem}

Una imagen de disco que contiene FAT32, ext4 o exFAT tiene una tabla de particiones
delante con la misma frecuencia que una que contiene XFS, y la detección lee el
desplazamiento cero, donde una imagen particionada tiene la *tabla*. Así que primero
se elige la partición, y al controlador se le entrega una vista de ella:

```hcl
share "photos" {
  image           = "/srv/disk.img"
  partition_label = "photos"          # what lsblk calls PARTLABEL
}
```

Tres formas de nombrarla, y **solo se puede dar una**: dos que no coincidieran
servirían la que el código probara primero:

| | |
|---|---|
| `partition = 2` | contando desde 1, como las muestran las herramientas de particionado |
| `partition_label = "photos"` | el nombre de partición GPT |
| `partition_uuid = "…"` | el GUID único GPT, el `PARTUUID` de Linux |

{{< callout type="error" >}}
**Un índice se mueve**

Un disco reparticionado, una herramienta que escribe las entradas en otro
orden, una imagen restaurada con una partición menos, y `partition = 2`
nombra otra cosa, **en silencio**, porque ahí se sigue encontrando un
sistema de archivos. Una etiqueta o un UUID nombran la propia partición, y
por eso los fstab dejaron de usar índices hace años. El índice está aquí
para las imágenes MBR, que no tienen ni lo uno ni lo otro.
{{< /callout >}}

Un recurso compartido que eligió una partición es de **solo lectura**, y lo indica al
arrancar: los desplazamientos del controlador son los de la partición, mientras que
el archivo de debajo es el disco entero, así que una escritura caería en ese
desplazamiento desde el inicio de la *imagen*, es decir, muy a menudo sobre la tabla
de particiones.

## Nombrar un sistema de archivos desactiva la detección {#naming-a-filesystem-turns-detection-off}

{{< callout type="error" >}}
**Esa es la intención, y ese es el riesgo**

La imagen se abre como *eso* o se rechaza. Una imagen FAT32 a la que se le
dice que es XFS no se convierte en un recurso compartido XFS; no arranca, e
indica como qué se le pidió abrirla.
{{< /callout >}}

`filesystem` se acepta también para los detectables, y entonces se comparan ambos: un
recurso compartido que dice `ext4` sobre una imagen FAT32 se rechaza con *the share
says ext4 and the image holds fat32*. Así es como un sitio rechaza una detección
errónea en vez de descubrirla más tarde.
[`check`]({{< relref "/configuration/check.md" >}}) marca con un asterisco un controlador
nombrado, porque esa fila no se reconoció: se afirmó.

## Un dispositivo, no solo una imagen {#a-device-not-only-an-image}

```hcl
share "photos" {
  image = "/dev/sda"        # a device node, not a file
}
```

**No interviene ningún privilegio.** Aquí no se monta nada —los controladores de
[`go-filesystems`](https://github.com/go-filesystems) leen ext4, xfs y los demás en
espacio de usuario—, así que no hay `mount(2)` y, por tanto, tampoco
`CAP_SYS_ADMIN`. Abrir un dispositivo es un `open(2)` corriente, regido por los
permisos del nodo:

| | propietario | modo | suficiente |
|---|---|---|---|
| Linux `/dev/sda` | `root:disk` | 0660 | pertenecer a `disk` |
| macOS `/dev/disk0` | `root:operator` | 0640 | pertenecer a `operator`, más Acceso total al disco |

Un rechazo indica qué grupo, en lugar de `permission denied`: la respuesta nunca es
`sudo`.

⛔ **Un recurso compartido de dispositivo es de solo lectura**, por la misma razón
que un recurso compartido que eligió una partición: escribir en un disco en uso no es
una decisión que tomar en su nombre.

{{< callout type="error" >}}
**El dispositivo se abre en exclusiva, y es una cuestión de corrección, no de prudencia**

Si el núcleo tiene montado un sistema de archivos a partir de él, la caché
de páginas del sistema de archivos contiene metadatos más recientes que el
dispositivo; así, un lector en bruto ve un bloque de directorio anterior a
una actualización junto a un bloque de inodo posterior a ella, un estado
que nunca existió en el disco en ningún momento. `O_EXCL` en un dispositivo
de bloques es la primitiva del propio núcleo para «nadie más, montajes
incluidos»; es la que usan `mkfs` y `fsck`. Un dispositivo en uso se
rechaza, por su nombre, con el motivo.
{{< /callout >}}

Dos cosas se midieron en lugar de suponerse:

- `Stat().Size()` es **0** para un dispositivo, así que la longitud se pregunta al
  propio dispositivo. Desplazarse hasta el final responde en Linux y devuelve **0 sin
  error** en macOS, por eso Darwin usa `DKIOCGETBLOCKCOUNT` en su lugar.
- Un nodo **en bruto** (`/dev/rdiskN`, u `O_DIRECT` en Linux) rechaza una lectura no
  alineada, y leer un campo de dos bytes en el desplazamiento 11 *es* precisamente
  analizar un BPB de FAT. Las lecturas de un dispositivo se redondean a bloques
  enteros; sin eso, `/dev/rdisk4` indicaba `unknown filesystem`.

## Un directorio, no solo una imagen {#a-directory-not-only-an-image}

Desde la v0.8.0, un recurso compartido puede servir un directorio del host —el
volumen de un contenedor, por ejemplo— en lugar de una imagen:

```hcl
share "photos" {
  directory = "/data/photos"
  allow     = ["@family"]
}
```

Es [go-filesystems/osfs](https://github.com/go-filesystems/osfs), que accede al árbol
a través de un **`os.Root`**: un `..` que sale de él, una ruta absoluta y un enlace
simbólico que lleva fuera los rechaza la raíz respaldada por el núcleo, no una
comprobación de cadenas. Un cliente puede *crear* un enlace a `/etc`; nada lo
seguirá. Un FIFO colocado en el árbol se rechaza en lugar de dejar que bloquee el
servidor.

{{< callout type="error" >}}
**Un recurso compartido no puede contener otro (desde la v0.17.0)**

Un recurso compartido de directorio cuyo árbol contiene la imagen o el
directorio de otro recurso compartido se **rechaza al arrancar**, y también
en la API de administración: quien pudiera usar el recurso exterior leería y
escribiría el interior sin tener permiso, NFS anónimo incluido. Lo mismo
ocurre con dos recursos compartidos sobre un mismo origen, salvo dos
particiones distintas de una misma imagen de disco.
{{< /callout >}}

{{< callout type="info" >}}
**Fuera del bloqueo de imagen**

Un controlador de imagen posee un único archivo y no promete nada sobre dos
llamadas a la vez, y por eso [cada imagen se envuelve en un bloqueo]({{< relref "/operations/locking.md" >}}).
Un árbol del host es del núcleo, que serializa lo que hay que serializar en
cada llamada y nada más; un mutex delante de cada archivo de cada cliente
sería lo más lento que puede hacer un servidor de archivos.
{{< /callout >}}

⛔ Un recurso compartido tiene una `image` **o** un `directory`, nunca ambos, y nunca
ninguno. `filesystem` y `partition` son para una imagen: un recurso compartido de
directorio que nombra uno se rechaza, porque indicarlo significa que el recurso
compartido debía ser una imagen. `read_only = true` funciona en un directorio igual
que en una imagen; un recurso compartido que lo indica y además enumera `writers` se
rechaza, porque `read_only` ganaría.

La capacidad que se comunica a un cliente es la del sistema de archivos en el que se
encuentra el árbol; una plataforma que no puede indicarla la deja a cero, que los
protocolos interpretan como «desconocida» y no como «llena».

## Qué hace SMB con una fecha que no puede almacenar {#what-smb-does-with-a-time-it-cannot-store}

Una fecha que establece un cliente y que el almacenamiento no puede conservar se
**ignora** por defecto. Desde la v0.42.0 un recurso compartido puede elegir, con
`unstorable_times`. No es almacenable una CreationTime distinta de la informada,
una ChangeTime enviada sola, o una fecha de acceso o de escritura en una imagen
cuyo controlador no puede establecer fechas (NTFS, APFS, ISO 9660, SquashFS; un
directorio del host y una imagen FAT, exFAT, ext4 o HFS+ las almacenan):

```hcl
share "photos" {
  directory        = "/data/photos"
  unstorable_times = "log"      # "ignore" (por defecto), "refuse" o "log"
}
```

`refuse` responde al cliente con un error y `log` escribe una línea con el
recurso y la ruta. Los casos exactos, según el tipo de recurso, están en
[Fechas reales de los archivos]({{< relref "/protocols/times.md#when-unstorable" >}}).
