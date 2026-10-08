---
title: "go-fileshare"
linkTitle: "Inicio"
type: docs
cascade:
  type: docs
description: "go-fileshare sirve imágenes de disco y directorios por SMB, NFS, WebDAV, SFTP y S3, con un único conjunto de usuarios y reglas por recurso compartido, desde una sola configuración."
---

**Una imagen de disco —o un directorio— servida por SMB, NFS, WebDAV, SFTP y S3:
los mismos usuarios, el mismo acceso por recurso compartido, desde un solo archivo
de configuración.** Go puro, `CGO_ENABLED=0`, un solo binario. Estas páginas describen
la [v0.25.0]({{< relref "/status.md" >}}).

```sh
go install github.com/go-fileshare/fileshare@latest

fileshare --image disk.img --user alice --password-file pw   # one image, now
fileshare --config /etc/fileshare.d                          # several, with users
```

La contraseña procede de un **archivo**, nunca de una opción: un argumento es visible
en la lista de procesos para cualquier usuario de la máquina.

## Por qué un solo programa {#why-one-program}

Qué protocolo transporta una imagen es una propiedad del cliente del otro extremo:
macOS y Windows recurren a SMB, un parque de Linux ya tiene NFS, un navegador o un
teléfono tiene HTTP. Ejecutar tres servidores, cada uno con su propio archivo de
configuración y su propia idea de quién es `alice`, es una forma de que dos de ellos
se equivoquen de manera sutil.

Así que hay una sola configuración, un solo conjunto de usuarios, un solo conjunto de
reglas por recurso compartido, y los protocolos son escuchas sobre ella.

## Lo que imprime al arrancar {#what-it-prints-at-startup}

```
smb    on 0.0.0.0:445 — photos and scratch
webdav on 0.0.0.0:8080 — photos and scratch
sftp   on 0.0.0.0:2222 — photos and scratch
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

Ese último párrafo resume la forma de todo el programa: una regla que la configuración
exige y un protocolo que no puede respetarla no se encuentran discretamente a medio
camino. Consulte [Lo que un protocolo puede prometer]({{< relref "/protocols/_index.md" >}}).

## Por dónde seguir {#where-to-go-next}

| | |
|---|---|
| [La configuración]({{< relref "/configuration/_index.md" >}}) | usuarios, grupos, recursos compartidos, bloques `serve` |
| [Recursos compartidos: imágenes, dispositivos, directorios]({{< relref "/configuration/shares.md" >}}) | qué se detecta y qué hay que declarar |
| [Usuarios, grupos y directorios]({{< relref "/configuration/identity.md" >}}) | archivos, SQL, LDAP, y lo que cada uno puede probar |
| [`check`]({{< relref "/configuration/check.md" >}}) | releer toda la configuración antes de reiniciar |
| [Protocolos]({{< relref "/protocols/_index.md" >}}) | lo que cada uno puede y no puede prometer |
| [La API de administración]({{< relref "/administration/_index.md" >}}) | recursos compartidos creados, concedidos y retirados de servicio sin reiniciar |
| [Volúmenes]({{< relref "/administration/volumes.md" >}}) | datasets ZFS, subvolúmenes btrfs y cuotas de proyecto XFS/ext4, creados a través de la API y servidos |
| [Salud y métricas]({{< relref "/administration/health.md" >}}) | `/healthz`, `/readyz`, `/metrics` |
| [Volver a leer el directorio]({{< relref "/administration/reload.md" >}}) | `reload`, `SIGHUP`, y lo que una eliminación hace a las sesiones abiertas |
| [TLS]({{< relref "/security/tls.md" >}}) | archivos o ACME; y por qué WebDAV se rechaza sin cifrar |
| [Qué revoca qué]({{< relref "/security/_index.md" >}}) | KRL, CRL, señales compartidas, y por qué cada una falla en modo cerrado |
| [Compilar solo lo que necesita]({{< relref "/operations/build-tags.md" >}}) | una etiqueta deja un protocolo completamente fuera |
| [Un proceso por protocolo]({{< relref "/operations/isolation.md" >}}) | `--isolate` |
| [Estado]({{< relref "/status.md" >}}) | lo que está verificado y lo que aún no está aquí |

## Licencia {#licence}

BSD-3-Clause.
