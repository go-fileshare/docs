---
title: "Volúmenes: almacenamiento que crea fileshare"
weight: 10
description: "Crear almacenamiento ZFS, btrfs, XFS y ext4 mediante el aprovisionador privilegiado de fileshare, y servir recursos compartidos a partir de él."
tags: [administración, volúmenes, zfs, btrfs, cuotas]
---

Desde la v0.21.0, la [API de administración]({{< relref "/administration/_index.md" >}}) puede **crear
el almacenamiento que sirve un recurso compartido** —un **dataset ZFS**, un
**subvolumen btrfs**, un **directorio XFS o ext4 bajo una cuota de proyecto**—, con
tamaño, propietario y seguimiento, y servir un recurso compartido a partir de él.
Solo en Linux. Ceph aún no está soportado.

Crear ese almacenamiento requiere `CAP_SYS_ADMIN` (quotactl, los ioctl de cuotas de
btrfs, `/dev/zfs`, mount(2)). `fileshare serve` analiza cinco protocolos de red para
desconocidos y no tiene ningún privilegio, así que la parte que sí lo tiene es un
segundo rol del mismo binario, `fileshare provisioner`, detrás de un socket unix y de
un protocolo pequeño y cerrado:

```
                         admin gRPC (unix socket 0600 + peer uid, or mTLS)
  operator ─────────────────────────────────────────────▶  fileshare serve
                                                             (unprivileged)
                                                                  │
                     provision gRPC (unix socket, SO_PEERCRED = fileshare's uid)
                                                                  ▼
                                                    fileshare provisioner
                                                    (CAP_SYS_ADMIN, CAP_CHOWN, CAP_FOWNER)
                                                                  │
                                    go-fsctl: zfs · btrfs · projquota (ioctls, no CLI)
```

Un binario en lugar de dos demonios: una versión que desplegar, una definición de
protocolo. El almacenamiento en sí lo gestiona [go-fsctl](https://go-fsctl.github.io/)
(Go puro, sin los comandos `zfs` ni `btrfs`). El diseño, y lo que cambió durante su
construcción, están en el
[`docs/volumes.md`](https://github.com/go-fileshare/fileshare/blob/v0.23.0/docs/volumes.md)
del repositorio de fileshare.

## Los dos procesos, configurados {#both-processes-configured}

`fileshare serve`, como el usuario `fileshare` (uid 990):

```hcl
# /etc/fileshare/fileshare.hcl
admin {
  listen          = "unix:///run/fileshare/admin.sock"
  state_file      = "/var/lib/fileshare/shares.json"
  source_roots    = ["/srv/fileshare/volumes"]
  provisioner     = "unix:///run/fileshare-provisioner/provisioner.sock"
  provisioner_uid = 0          # who must answer on that socket; 0 is the default
  allowed_uids    = [990]      # who may call this API
}
```

| clave | |
|---|---|
| `provisioner` | el socket del aprovisionador, `unix:///ruta`. Sin él, toda llamada de volúmenes responde `FAILED_PRECONDITION` y no se crea ningún recurso compartido a partir de un volumen |
| `provisioner_uid` | el uid con el que debe ejecutarse el proceso que responde en ese socket, leído del núcleo (`SO_PEERCRED`) en cada conexión: un socket que otro haya vinculado no recibe nada. Si no se establece, es 0 |
| `allowed_uids` | opcional, solo para socket unix: los únicos uid cuyas llamadas se responden; cualquier otro recibe `PERMISSION_DENIED`, y queda auditado. Solo puede **restringir** lo que permite el modo 0600 del socket (su propietario y root): root, por ejemplo, puede quedar sin permiso para llamar. Se rechaza por TCP, donde la comprobación es el certificado de cliente |

La ruta de un volumen debe estar bajo `source_roots`, como cualquier otro origen.

`fileshare provisioner`, como root:

```hcl
# /etc/fileshare-provisioner.hcl
provisioner {
  listen     = "unix:///run/fileshare-provisioner/provisioner.sock"
  client_uid = 990
  group      = "fileshare"
  max_volume = "10T"
  state_file = "/var/lib/fileshare-provisioner/volumes.json"

  parent "tank" {
    zfs  = "tank/fileshare"
    root = "/srv/fileshare/volumes/tank"
  }
  parent "fast" { btrfs = "/srv/fileshare/volumes/fast" }
  parent "plain" {
    xfs         = "/srv/fileshare/volumes/plain"
    project_ids = "100000-199999"
  }
}
```

| clave | |
|---|---|
| `listen` | `unix://<ruta absoluta>` y nada más. El socket se crea con 0660, propiedad del usuario del aprovisionador (root) y de `<group>`, y debe estar en un directorio en el que nadie más que el aprovisionador pueda escribir: el **suyo** (`/run/fileshare-provisioner`), no el de fileshare |
| `client_uid` | el único uid al que se responde: el de fileshare. **0 se rechaza** |
| `group` | es propietario de la raíz de cada volumen (`root:<group>`, modo 2770). fileshare debe pertenecer a él: escribe en un volumen a través del grupo, nunca como propietario; el propietario de un directorio puede borrar su id de proyecto sin privilegios, y así salirse de una cuota XFS/ext4 |
| `max_volume` | la cuota más grande que puede tener un volumen (`"10T"`, `"500G"`, `"1048576"`); por encima, `OUT_OF_RANGE` |
| `state_file` | lo que ha creado este proceso —ids de subvolúmenes btrfs, ids de proyecto XFS/ext4—, para que tras un reinicio sepa qué es suyo |
| `parent "<id>"` | dónde se pueden crear volúmenes, con el id que nombra una petición. Exactamente uno de `zfs` (el dataset del que los volúmenes son hijos, con `root` donde se montan), `btrfs`, `xfs` o `ext4` (el directorio en el que se crean) |
| `project_ids` | XFS/ext4: el rango de ids de proyecto que este padre puede asignar |
| `enable_quota` | btrfs: permite al aprovisionador activar las cuotas cuando están desactivadas. Sin ello, un padre btrfs con las cuotas desactivadas detiene el arranque: los qgroups ralentizan cada commit a medida que se multiplican las instantáneas, y esa es una decisión del operador |

El archivo del aprovisionador no contiene nada más que su bloque `provisioner`: el
propio archivo de fileshare entregado por error se rechaza en lugar de leerse a
medias.

Un volumen es siempre `<raíz del padre>/<nombre>`: el dataset `tank/fileshare/<name>`
montado en `/srv/fileshare/volumes/tank/<name>` (con `refquota`), un subvolumen btrfs
limitado por su qgroup, un directorio XFS o ext4 con un id de proyecto del rango y un
límite estricto de bloques. XFS y ext4 necesitan la opción de montaje `prjquota`
(ext4, además, las funcionalidades `project,quota`).

## La pareja de unidades systemd {#the-systemd-pair}

El aprovisionador:

```ini
# fileshare-provisioner.service
[Service]
ExecStart=/usr/local/bin/fileshare provisioner -c /etc/fileshare-provisioner.hcl
RuntimeDirectory=fileshare-provisioner
RuntimeDirectoryMode=0755
StateDirectory=fileshare-provisioner
StateDirectoryMode=0700
CapabilityBoundingSet=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
# Run as root, or as a dedicated user with:
# AmbientCapabilities=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
NoNewPrivileges=yes
SystemCallFilter=@system-service @mount quotactl quotactl_fd
SystemCallArchitectures=native
LockPersonality=yes
RestrictRealtime=yes
# NOT ProtectSystem= nor ReadWritePaths=: they give the service its own
# mount namespace, and the ZFS volumes it mounts would be seen by nobody
# else. And no Landlock: a Landlock-restricted thread cannot mount(2).
```

El servidor, sin ninguna capability:

```ini
# fileshare.service
[Unit]
Requires=fileshare-provisioner.service
After=fileshare-provisioner.service

[Service]
User=fileshare
Group=fileshare
ExecStart=/usr/local/bin/fileshare serve -c /etc/fileshare/fileshare.hcl
RuntimeDirectory=fileshare
StateDirectory=fileshare
# No capability at all: CAP_SYS_RESOURCE would let it past an ext4 quota,
# and fileshare refuses to serve volumes with it.
CapabilityBoundingSet=
AmbientCapabilities=
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/srv/fileshare/volumes /var/lib/fileshare
```

`ProtectSystem=` no plantea problemas para el servidor: quien monta es el
aprovisionador, y los montajes se propagan hacia el espacio de nombres del servidor,
no fuera de él, así que un volumen ZFS montado después de que arrancara el servidor
le resulta visible.

{{< callout type="info" >}}
**Ejemplos, no unidades probadas**

Ninguna de las dos unidades se ejercita en la CI: el job de extremo a
extremo ejecuta ambos procesos con `sudo`.
{{< /callout >}}

{{< callout type="warning" >}}
**Actualice primero el aprovisionador**

El aprovisionador rechaza un mensaje que lleve un campo que su versión no
define —cierra la conexión—, así que `fileshare serve` y
`fileshare provisioner` ejecutan la misma versión.
{{< /callout >}}

## Las llamadas de volúmenes de la API de administración {#the-admin-apis-volume-calls}

`fileshare serve` las transmite al aprovisionador
([`admin.proto`](https://github.com/go-fileshare/fileshare/blob/v0.23.0/proto/fileshare/admin/v1/admin.proto)):

| | |
|---|---|
| `ListParents` | dónde se pueden crear volúmenes: el id de cada padre, su tipo, su raíz, el espacio libre y total, si admite instantáneas; y `max_volume_bytes` |
| `CreateVolume` | un volumen de `quota_bytes` (0 se rechaza). Idempotente por nombre: la misma cuota responde con el volumen existente (`created` a false), otra cuota da `ALREADY_EXISTS` |
| `ResizeVolume` | cambia la cuota, en uno u otro sentido; nunca por debajo de lo usado (`FAILED_PRECONDITION`) |
| `SnapshotVolume` | una instantánea con nombre, de solo lectura: ZFS y btrfs; `UNIMPLEMENTED` en XFS y ext4 |
| `GetVolume`, `ListVolumes` | cada volumen con su cuota, lo que usa, sus instantáneas y **los recursos compartidos que lo usan** |
| `DeleteVolume` | se rechaza mientras un recurso compartido lo usa: elimine antes el recurso compartido. Un volumen con datos o instantáneas necesita `destroy_data` |
| `CreateShare` con `volume { parent, name }` | un recurso compartido servido a partir del volumen, como directorio |

`DeleteShare` nunca elimina un volumen. «Los recursos compartidos que lo usan» son los
recursos compartidos creados a partir de él **y** cualquier recurso compartido
—también de los archivos de configuración, servido, desactivado o no disponible—
cuya imagen o directorio esté dentro de él.

Un nombre cumple `^[a-z0-9][a-z0-9_-]{0,62}$`, y una petición nunca lleva una ruta:
nombra un `parent` de la propia configuración del aprovisionador.

Con `reflection = true` en el bloque `admin`, `grpcurl` puede hacer las llamadas. El
socket es 0600 y el `allowed_uids` de arriba solo nombra el uid de fileshare, así que
ejecútelo como ese usuario:

```sh
S=/run/fileshare/admin.sock
A=fileshare.admin.v1.AdminService

sudo -u fileshare grpcurl -plaintext -unix $S $A/ListParents

# a 32 GiB volume on the XFS parent
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "34359738368"}' \
  $A/CreateVolume

# a share served from it
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"name": "projects",
       "volume": {"parent": "plain", "name": "projects"},
       "grants": [{"subject": {"group": "staff"}, "access": "ACCESS_WRITE"}]}' \
  $A/CreateShare

# grow it to 64 GiB
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "68719476736"}' \
  $A/ResizeVolume
```

### Códigos de estado {#status-codes}

Los códigos del aprovisionador se devuelven tal como los dio: `INVALID_ARGUMENT` (un
nombre fuera de la gramática, un padre desconocido, una cuota de 0), `OUT_OF_RANGE`
(por encima de `max_volume`), `RESOURCE_EXHAUSTED` (la cuota, o el crecimiento de un
redimensionado, supera lo que el padre tiene libre ahora; o el rango de ids de
proyecto se ha agotado), `ALREADY_EXISTS`, `NOT_FOUND`, `FAILED_PRECONDITION`,
`UNIMPLEMENTED`.

Tres son propios de fileshare:

- ningún `provisioner` configurado: `FAILED_PRECONDITION`;
- el aprovisionador rechaza el uid de este servidor: también `FAILED_PRECONDITION`;
  es un error de despliegue (su `client_uid` es incorrecto), no algo que el llamante
  de administración no tenga permitido;
- el aprovisionador no responde: `UNAVAILABLE`.

## Lo que se comprueba antes de servir un volumen {#what-is-checked-before-a-volume-is-served}

fileshare pregunta al aprovisionador dónde está el volumen, y luego comprueba por sí
mismo:

- que la ruta es limpia, absoluta y —una vez resuelta— está bajo `source_roots`;
- que es un directorio;
- que `statfs` indica el sistema de archivos del tipo (`ZFS_SUPER_MAGIC`,
  `BTRFS_SUPER_MAGIC`, `XFS_SUPER_MAGIC`, `EXT4_SUPER_MAGIC`);
- **ZFS**: que es un punto de montaje, es decir, que su dispositivo no es el de su
  padre. El directorio de un dataset sin montar aceptaría escrituras sin ninguna
  cuota. (No se verifica qué dataset está montado ahí: la respuesta del
  aprovisionador no lo indica.)
- **btrfs**: que es la raíz de un subvolumen; un directorio corriente bajo el padre
  no tiene qgroup;
- **XFS y ext4**: que lleva un id de proyecto distinto de cero que heredan sus nuevos
  archivos; un directorio fuera de todo proyecto se sirve sin límite.

El archivo de estado guarda el **nombre** del volumen, no su ruta, y cada arranque
vuelve a preguntar y a comprobar. Un volumen que ha desaparecido, que no supera una
comprobación, o cuyo aprovisionador no responde en 10 s deja su recurso compartido
**definido y sin servir** —se indica al arrancar, en
[`fileshare check`]({{< relref "/configuration/check.md" >}}) y en el campo `unavailable`
del recurso compartido— mientras el resto del servidor arranca. `EnableShare` lo
vuelve a intentar.

{{< callout type="error" >}}
**`fileshare serve` se niega a servir volúmenes como root o con `CAP_SYS_RESOURCE`**

ext4 permite a cualquiera de los dos escribir más allá de una cuota de
proyecto (fs/quota/dquot.c, `ignore_hardlimit`): root escribió 16 MiB en un
proyecto de 8 MiB en la CI de go-fsctl/projquota. XFS no tiene esa
excepción, pero la regla es una sola para el proceso, en todos los tipos:
fileshare lee su uid efectivo y `CapEff`/`CapPrm` en `/proc/self/status`
(una capability permitida está a un `capset(2)` de ser efectiva), y un
estado que no puede leer cuenta como si la tuviera. Se niega a **servir**,
no a las llamadas de volúmenes: crear almacenamiento como root es inofensivo,
escribir en él como root no lo es. `fileshare check` indica cuál es el caso.
Por eso el aprovisionador rechaza `client_uid = 0`, y la unidad del servidor
de arriba tiene un `CapabilityBoundingSet=` vacío.
{{< /callout >}}

## Un recurso compartido lleno {#a-full-share}

XFS indica un proyecto lleno con `ENOSPC`; ext4, btrfs y ZFS indican `EDQUOT`. Desde
la v0.21.1, todos los protocolos responden «lleno»:

| protocolo | sin espacio (`ENOSPC`) | una cuota (`EDQUOT`) |
|---|---|---|
| WebDAV | `507 Insufficient Storage` | `507 Insufficient Storage` |
| NFS | `NFS3ERR_NOSPC` (28) | `NFS3ERR_DQUOT` (69), como el RFC 1813 y el nfsd de Linux |
| SFTP | `SSH_FX_FAILURE`, "no space left on device (the share is full)"; la versión 3 no tiene código para ello | lo mismo |
| SMB | `STATUS_DISK_FULL` (0xC000007F) | `STATUS_DISK_FULL`, como hace Samba ("Windows apps need this, not NT_STATUS_QUOTA_EXCEEDED") |
| S3 | servido en solo lectura | servido en solo lectura |

En la v0.21.0, SMB respondía `STATUS_ACCESS_DENIED` y NFS respondía a una cuota con
`NFS3ERR_NOSPC`.

Esto se aplica a todo [recurso compartido de directorio]({{< relref "/configuration/shares.md" >}}),
no solo a los volúmenes; consulte la [nota de actualización]({{< relref "/status.md" >}}).

## El tamaño que ve un cliente {#the-size-a-client-sees}

Cada protocolo pregunta al servidor el tamaño de un recurso compartido en cada
consulta: `FSSTAT` en NFS, `FileFsFullSizeInformation` en SMB y las propiedades de
cuota de la RFC 4331 en WebDAV.

- **ZFS, XFS y ext4**: el tamaño es lo que dice `statfs` dentro del volumen, al que el
  núcleo responde con la cuota (`refquota` en ZFS, la cuota de proyecto en XFS y ext4).
- **btrfs** (desde fileshare v0.23.0): el `statfs` de btrfs ignora los qgroups y
  declara el sistema de archivos entero, así que el servidor usa las cifras del
  aprovisionador. El tamaño es la cuota del volumen, y el espacio libre es la cuota
  menos los bytes referenciados del qgroup, o el espacio libre del sistema de archivos
  si es menor. El servidor consulta al aprovisionador en segundo plano, como mucho una
  vez cada 5 segundos mientras haya clientes preguntando, y mientras tanto responde con
  las últimas cifras. btrfs actualiza la cuenta de un qgroup al confirmar una
  transacción (cada 30 segundos por defecto), así que el espacio libre que ve un cliente
  puede ir con ese retraso respecto a las escrituras. Antes de la v0.23.0, un volumen
  btrfs mostraba el sistema de archivos entero; la cuota se aplicaba igualmente.

El espacio usado procede del aprovisionador (`used_bytes`): leer el uso de un
proyecto XFS o ext4 requiere `CAP_SYS_ADMIN`, que fileshare no tiene.

## Lo que promete el aprovisionador {#what-the-provisioner-promises}

- **Un único uid, leído del núcleo.** El par se identifica con `SO_PEERCRED` cuando
  se conecta, y cualquiera que no sea `client_uid` —root incluido— recibe
  `PERMISSION_DENIED` antes de que se decodifique un solo byte de la petición.
- **Un conjunto cerrado de verbos.** Un método fuera de su servicio, o un mensaje que
  no se decodifica o que lleva un campo que no define, no recibe respuesta: la
  conexión se cierra.
- **Nunca toca lo que no ha creado.** ZFS: la propiedad `fileshare:volume` propia del
  dataset, establecida en el dataset, ya que las propiedades de usuario se heredan y
  una marca heredada no es propia. btrfs: el id y el uuid del subvolumen registrados
  en `state_file`. XFS/ext4: un id de proyecto dentro del rango del padre *y*
  registrado. Se comprueba antes de cada redimensionado, instantánea y eliminación.
- **La eliminación rechaza datos e instantáneas** salvo que se establezca
  `destroy_data`, y se registra antes de que ocurra.

## Medido {#measured}

En la CI de fileshare, como root sobre pools reales y sistemas de archivos sobre
dispositivos loop, con `fileshare serve` ejecutándose como `nobody` y dirigido a
través de la API de administración: las escrituras por WebDAV y por SFTP se detienen
en la cuota de un volumen de 32 MiB en los cuatro tipos.

## Todavía no {#not-yet}

- **Ceph.** Las cuotas de CephFS y RBD llegarán en una fase posterior.
