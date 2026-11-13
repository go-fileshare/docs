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

## Las imágenes HFS+ son de solo lectura salvo que un recurso lo acepte {#hfsplus-images-are-writable}

Desde v0.44.0 (hfsplus v0.9.0), un recurso cuya imagen contiene HFS+ puede
abrirse para escritura, como FAT, exFAT y ext4: un cliente puede crear, cambiar y
borrar archivos, y las fechas, modos y propietarios que establece se guardan.
Antes, una imagen HFS+ se abría en solo lectura.

**Desactivado por defecto desde v0.45.1, y lo sigue estando.** hfsplus v0.9.0 no
resistía a los fallos (mantenía toda la imagen en memoria, la reescribía entera
en cada operación y nunca marcaba el volumen como en uso), así que v0.45.1
desactivó la escritura, que v0.44.0 y v0.45.0 permitían por defecto (hasta
256 MiB). **Desde v0.46.0 (hfsplus v0.10.0)**, un volumen HFS+ escribible se
escribe **en el sitio**:

- **Sin coste de memoria.** La imagen no se copia en memoria (una caché de lectura
  de 8 MiB); solo se escriben los sectores que cambia cada confirmación. El
  límite de 64 GiB desaparece.
- **Un fallo es visible.** Cada confirmación marca el volumen como en uso: el bit
  de desmontado y el bit de journal se borran en la primera escritura de la
  cabecera y se restauran al final, de modo que `fsck_hfs -q` ve un fallo,
  también en volúmenes con journal.
- **Una escritura fallida detiene el volumen.** Tras un vaciado fallido, el
  volumen rechaza las escrituras (`ErrVolumeFailed`) hasta que se reabre el
  recurso.

Sigue siendo **opcional**, por lo que hfsplus aún no ha terminado: una escritura
pequeña y duradera cuesta 3 fsync por confirmación y es unas 3 veces más lenta
que en macOS (go-filesystems/hfsplus#35), las pruebas de fallos aún no están
reforzadas (go-filesystems/hfsplus#37), y tras un fallo un volumen con journal se
monta sin su journal hasta `diskutil enableJournal`. El valor por defecto podrá
volver cuando se corrija. Haga antes una copia de la imagen. Un recurso lo
acepta con `hfsplus_writable_max`, que por defecto es `"0"` (nunca; un número de
bytes, o uno con K, M, G o T):

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"   # aceptar
}
```

Desde v0.46.0 el tamaño ya no limita la memoria. Sigue siendo un tamaño para que
una configuración escrita para v0.44.0 a v0.45.1 signifique lo que significaba
(un recurso que decía `"256M"` no empieza a escribir una imagen de 2 TiB); un
valor grande como `"1T"` es la simple aceptación.

Con el valor por defecto, por encima del tamaño o con tamaño desconocido, el
recurso se abre en **solo lectura**, aún con lecturas por rangos y fechas, y el
servidor escribe una línea con la imagen y el motivo, por ejemplo sin ningún
ajuste:

```text
/srv/mac.img is an HFS+ volume and hfsplus_writable_max is 0, the default: HFS+ images are written only when a share sets a size, because a durable small write is about 3x slower than on macOS (go-filesystems/hfsplus#35), the crash tests are not finished (go-filesystems/hfsplus#37), and after a crash a journaled volume mounts without its journal until diskutil enableJournal, so mac is read-only
```

y por encima del tamaño:

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes), so mac is read-only
```

**Una recarga sigue el ajuste (desde v0.46.0).** Cuando una recarga cambia si una
imagen HFS+ se escribe (ajuste quitado, añadido o movido a uno y otro lado del
tamaño de la imagen), la imagen se vuelve a abrir: quitar la aceptación deja el
recurso en solo lectura al instante, añadirla lo hace escribible. Antes de
v0.46.0 se conservaba el controlador abierto, así que un recurso seguía siendo
escribible tras quitar su aceptación hasta que el servidor se reiniciaba. Pasar
de un tamaño escribible a otro no cambia nada.

Un volumen **con journal** (creado por `hdiutil create -fs "Journaled HFS+"` o por
macOS) solo se escribe cuando su journal está vacío, que es lo que deja un
desmontaje limpio. Un journal con transacciones, o uno que no se puede
verificar, sería reproducido por macOS sobre lo que escribe el servidor, así que
el controlador lo rechaza y **el recurso se abre en solo lectura en su lugar**. El
servidor sigue arrancando; al abrir el recurso escribe una línea con la imagen,
el recurso y el motivo:

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

Para que un recurso así sea escribible, hay que reproducir antes el journal:
montar la imagen en un Mac y expulsarla limpiamente, o ejecutar `fsck_hfs` sobre
ella. Mientras se escribe un volumen con journal, su bit de journal está
borrado; tras un fallo sigue borrado, así que el volumen se monta sin journal
hasta `diskutil enableJournal`. Un recurso con `read_only = true`, un dispositivo
y una partición nunca intentan la apertura para escritura.

La imagen se abre una segunda vez por su nombre para escribir (hfsplus recibe una
ruta), y solo si ese nombre sigue siendo el mismo archivo que tiene el recurso;
si no, el recurso se abre en solo lectura y lo dice.

**Otro escritor.** Un segundo escritor sobre la misma imagen la corrompería. La
apertura para escritura toma un bloqueo consultivo exclusivo `flock` sobre la
imagen (Linux, macOS y los BSD); si otro proceso lo tiene, por ejemplo un segundo
fileshare sobre la misma imagen, el recurso se abre en solo lectura y lo dice. El
bloqueo es consultivo: un montaje de macOS no lo toma, y en otras plataformas no
hay bloqueo.

⛔ **No escriba una imagen montada**, ni con este servidor ni con otra cosa:
ninguna de las dos partes conoce los cambios de la otra. Sirva una imagen que
nada más tenga abierta.

## Una imagen escribible que el SO tiene montada se sirve en solo lectura {#an-image-the-os-has-mounted}

Desde v0.45.0, antes de abrir para escritura **cualquier** recurso compartido de
imagen escribible (FAT32, exFAT, ext4, UFS, btrfs, XFS, ZFS, HFS+, sea cual sea
el controlador), el servidor pregunta al sistema operativo si tiene esa imagen
asociada o montada. Si es así, el recurso se abre en **solo lectura**, se marca
como tal, y el servidor escribe una línea con la imagen, el dispositivo y el
montaje del SO, y el recurso:

```text
/srv/disk.img is in use by the OS (/dev/loop3, mounted at /mnt/disk), and a second writer would corrupt it, so disk is read-only
```

El `flock` consultivo descrito arriba no puede lograrlo: el SO no toma ningún
bloqueo al montar una imagen. La imagen se identifica por dispositivo e inodo,
nunca comparando cadenas de ruta, de modo que también se detecta un enlace
simbólico o físico al archivo montado.

| SO | Cómo se pregunta |
| --- | --- |
| Linux | `/sys/block/loop*/loop/backing_file` de cada dispositivo loop (lo que crean `mount -o loop`, `losetup` y udisks); el punto de montaje sale de `/proc/self/mountinfo`. Un dispositivo loop asociado cuenta aunque no haya nada montado en él. |
| macOS | `hdiutil info -plist`, el `image-path` de cada imagen asociada, una llamada por carga, con un límite de 15 s. Una imagen asociada con `-nomount` cuenta. |
| FreeBSD, otros BSD, Windows | **No se puede detectar.** El recurso es escribible, como antes. |

**Cuando falla la propia comprobación** (no hay `/sys/block` en un contenedor,
falta `hdiutil` o supera el límite, la salida no es una lista de propiedades),
nadie sabe si la imagen está montada. Decide `image_mount_unknown`:

```hcl
share "disk" {
  image               = "/srv/disk.img"
  image_mount_unknown = "read_only"   # el valor por defecto; "write" la sirve escribible igualmente
}
```

El valor por defecto es `read_only`: un recurso de solo lectura es un estado
visible con una línea de registro que nombra su causa, y una persona puede
corregirlo; un recurso escribible bajo un montaje corrompe los datos de ambos
lados, y nadie lo ve hasta más tarde. `write` es para un host donde la
comprobación no puede funcionar (un contenedor sin sysfs) y nadie monta
imágenes. Una imagen que se sabe montada por el SO es de solo lectura diga lo que
diga este ajuste. Una ruta de respaldo que la comprobación no puede leer
(permiso denegado, un bucle de enlaces simbólicos) es «no se pudo juzgar», no
«otra imagen»: va al mismo ajuste. Solo se omite una ruta que no existe. En Linux,
un dispositivo loop cuyo archivo de respaldo se borró se identifica mediante el
registro del propio kernel (`LOOP_GET_STATUS64`) si el proceso puede abrir
`/dev/loopN`; si no, no se identifica.

**Lo que la comprobación no ve**

- **Un montaje hecho después de que el recurso abrió la imagen.** La
  comprobación se ejecuta al abrir una imagen: al arrancar, para un recurso
  creado mediante la API de administración, y cuando una recarga reconstruye los
  recursos (un cambio de personas o listas) y encuentra la imagen montada desde
  entonces; ese recurso recibe entonces una apertura propia en solo lectura. Una
  recarga que no cambia nada no vuelve a mirar, y nada vigila un
  `hdiutil attach` o `mount -o loop` posterior. Detenga el recurso o reinicie el
  servidor antes de montar una imagen que sirve como escribible.
- **Montajes que no pasan por un dispositivo loop ni por `hdiutil`:**
  `qemu-nbd -c` (un dispositivo nbd), montajes FUSE (`fuse2fs`, `guestmount`) y
  una máquina virtual u otro programa que tenga la imagen abierta.
- Otros sistemas operativos (véase la tabla) y un dispositivo loop asociado en
  otro espacio de nombres de montaje.
- Un recurso puesto en solo lectura a causa de un montaje sigue en solo lectura
  tras una recarga aunque el montaje haya desaparecido; reinícielo para abrirlo
  de nuevo para escritura.

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
