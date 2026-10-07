---
title: "Qué revoca qué"
linkTitle: "Seguridad"
weight: 40
description: "Qué mecanismo retira cada tipo de credencial que fileshare acepta, y por qué todos ellos fallan en modo cerrado."
tags: [seguridad, revocación]
---

Una credencial se comprueba cuando se presenta. Retirarle después el acceso a una
persona —un miembro del personal que se fue, una cuenta desactivada en
go-authn/bridge, todo un IdP apagado— tiene que alcanzar **todas** las credenciales
que recibió, y las sesiones que estas ya abrieron. Cada tipo de credencial se alcanza
aquí con un mecanismo distinto, porque cada una se emite y se comprueba de forma
distinta:

| credencial | protocolo | qué la retira | añadido en |
|---|---|---|---|
| una contraseña, un hash NT o una contraseña de aplicación en un directorio `users` (SQL, LDAP) | SMB, WebDAV, S3 | la fila eliminada y luego una [recarga del directorio]({{< relref "/administration/reload.md" >}}): una nueva generación, y las conexiones que abrió, cerradas | v0.10.0 |
| un permiso sobre un recurso compartido | todos los protocolos | `Revoke` de la [API de administración]({{< relref "/administration/_index.md" >}}), o `DisableShare`: una nueva generación, conexiones cerradas | v0.8.0 |
| un certificado SSH firmado por la CA del proveedor | SFTP | su [KRL]({{< relref "/security/revocation-lists.md" >}}), comprobada al iniciar sesión **y en cada operación**; y las [señales compartidas]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |
| un certificado OpenPubkey (opkssh) | SFTP | ninguna lista puede nombrarlo; las [señales compartidas]({{< relref "/security/shared-signals.md" >}}), y `opkssh_max_age` | v0.13.0 |
| un token de acceso | WebDAV | ninguna lista puede nombrarlo; las [señales compartidas]({{< relref "/security/shared-signals.md" >}}) | v0.13.0 |
| un certificado de cliente X.509 que nombra a una persona | NFS | su [CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}), en la siguiente llamada; y las [señales compartidas]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |

Sin el mecanismo de la tercera columna, una credencial sigue siendo válida hasta que
caduca, y para un certificado o un token ese margen es toda su vida útil.

## Todos fallan en modo cerrado {#every-one-of-them-fails-closed}

Una lista de revocación que no se puede descargar, una copia más antigua que su
`max_age`, una CRL que ha superado su propio `NextUpdate`, un transmisor de señales
compartidas del que no se sabe nada dentro de `max_age`: nada de esto se interpreta
como «no hay nada revocado». Significa «este servidor no lo sabe», y las credenciales
que ese mecanismo gobierna se **rechazan** hasta que vuelva a saberlo.

{{< callout type="error" >}}
**El precio es la disponibilidad, y se paga a propósito**

Una comprobación de revocación que deja pasar todo cuando la lista es
inalcanzable es la que derrota un atacante capaz de bloquear la lista; por
eso se dijo del OCSP en modo «soft-fail» de los navegadores que era «un
cinturón de seguridad que se rompe cuando choca». Así que la lista, o el
transmisor, debe servirse con la misma fiabilidad que los inicios de sesión
que gobierna, y las métricas que indican que se está quedando obsoleta son
las que hay que [vigilar con alertas]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

Una copia solo se sustituye por una lista que se ha podido analizar: una lista que
llega dañada deja en su sitio la última válida, que sigue contando para `max_age`
desde el momento en que **ella** se descargó. Y una lista se descarga **solo por
HTTPS** —una lista que un atacante en el camino puede sustituir no revoca nada—, así
que una URL en HTTP plano se rechaza al arrancar.

## Transporte {#transport}

Las credenciales mismas no deben cruzar una red sin cifrar. Para eso está
[TLS]({{< relref "/security/tls.md" >}}): WebDAV, S3 y NFS sobre TLS, un certificado a partir de archivos o de ACME,
y —desde la v0.9.0— **WebDAV con contraseñas rechazado sin cifrar**.
