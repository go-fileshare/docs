---
title: "Un proceso por protocolo"
weight: 20
description: "Ejecutar cada protocolo en su propio proceso con --isolate, y las reglas que lo mantienen seguro."
tags: [operación, aislamiento]
---

```sh
fileshare --config /etc/fileshare.d --isolate
```

```
smb    on 0.0.0.0:445  — process 19810, serving public, photos and scratch
webdav on 0.0.0.0:8080 — process 19811, serving public and photos
nfs    on 0.0.0.0:2049 — process 19812, serving public
```

El proceso padre crea las escuchas y luego se ejecuta **a sí mismo** (exec) una vez
por protocolo, entregando a cada hijo solo los recursos compartidos que ese protocolo
puede servir.

## Comprobado con `lsof`, no leyendo el código {#checked-with-lsof-not-by-reading-the-code}

```
smb     (pid 19810) has open: scratch.img photos.img
webdav  (pid 19811) has open: photos.img
nfs     (pid 19812) has open: photos.img
```

La imagen escribible está abierta en **exactamente un proceso**, y el hijo WebDAV
nunca la abre.

Eso es lo que las [etiquetas de compilación]({{< relref "/operations/build-tags.md" >}}) no pueden
dar: un panic o un heap agotado en un protocolo tumba un solo protocolo, cada hijo
puede confinarse con lo que ofrezca el sistema operativo, y un fallo en un analizador
no puede alcanzar una imagen que ese proceso nunca abrió.

## Puertos privilegiados {#privileged-ports}

En Unix, el padre pasa la escucha ya vinculada al hijo, de modo que **un puerto
privilegiado funciona con hijos sin privilegios**: el padre se vincula al 445 y el
hijo nunca necesita el privilegio.

Windows no tiene `ExtraFiles`, así que allí el hijo se vincula él mismo a la
dirección; el aislamiento es el mismo, la parte del puerto privilegiado no.

## La regla que lo hace honesto {#the-rule-that-makes-it-honest}

Un hijo abre la imagen él mismo, así que una imagen servida *en escritura* por dos
protocolos serían dos controladores sobre un mismo archivo **sin ningún bloqueo entre
ellos**, justo lo que el [bloqueo compartido]({{< relref "/operations/locking.md" >}}) impide
dentro de un solo proceso.

Eso se rechaza, y el rechazo indica cómo corregirlo:

```
scratch is writable over smb and webdav, and one process per protocol means
that many drivers writing one file with no lock between them. Say
`protocols = ["smb"]` on the share, or make it read_only, or do not isolate.
```

`protocols = [...]` en un recurso compartido es útil por sí mismo, no solo con
`--isolate`: un recurso compartido declarado solo SMB es un recurso compartido del
que nunca se informa al proceso WebDAV. Un bloque `serve` que acabaría sin servir
nada también se rechaza, antes de abrir ninguna imagen, nombrando los recursos
compartidos que se le retiraron y por qué.

## Todavía no con un bloque `admin` o `metrics` {#not-with-an-admin-or-a-metrics-block-yet}

`--isolate` junto con un bloque [`admin`]({{< relref "/administration/_index.md" >}}) o
[`metrics`]({{< relref "/administration/health.md" >}}) se rechaza: los hijos abren las
imágenes y el padre no abre nada, así que no hay un único proceso al que se pueda
aplicar un cambio de la API, ni cuya disponibilidad pueda consultar una sonda.
