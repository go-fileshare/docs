---
title: "Lo que un protocolo puede prometer"
linkTitle: "Protocolos"
weight: 20
description: "Por qué cada protocolo puede o no puede saber quién pregunta, y lo que eso implica para los recursos compartidos que sirve."
tags: [protocolos]
---

Los protocolos no coinciden en lo único que necesita el control de acceso: si el
servidor puede saber **quién** pregunta.

| | |
|---|---|
| **[SMB]({{< relref "/protocols/smb.md" >}})** | NTLMv2. La contraseña nunca cruza la red, y el recurso compartido indica a quien solo puede leer que es así, en la máscara de acceso, antes de que lo intente. |
| **[WebDAV]({{< relref "/protocols/webdav.md" >}})** | HTTP Basic, sobre [TLS]({{< relref "/security/tls.md" >}}): rechazado sin cifrar en una dirección alcanzable, salvo que `plaintext = true` indique que un proxy termina TLS delante. Un recurso compartido que una persona no puede usar responde 404, no 403: no se confirma que exista. |
| **[SFTP]({{< relref "/protocols/sftp.md" >}})** | Una **clave pública**, o un **certificado SSH** de una autoridad en la que usted confía: el servidor nunca guarda el secreto y, con un certificado, el acceso de una persona se emite y caduca en otro lugar. Sin contraseña: un cliente que la pide hace justo lo que las claves existen para evitar. |
| **[OIDC]({{< relref "/protocols/webdav.md" >}})** (sobre WebDAV) | Un **token bearer** firmado por un proveedor de identidad. Verificado por [go-authn/oidc](https://github.com/go-authn/oidc): firma, emisor, audiencia, caducidad. Ningún otro protocolo de los aquí descritos tiene dónde ponerlo: S3 firma con SigV4, que no tiene campo para un token bearer. Por [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}), la palabra del proveedor llega en cambio dentro de un certificado. |
| **[S3]({{< relref "/protocols/s3.md" >}})** | **SigV4**, en cabecera o prefirmado. El secreto se demuestra calculando un HMAC y nunca cruza la red; por eso, como con NTLMv2, el directorio debe GUARDAR la contraseña, y no solo comprobarla. Un recurso compartido es un bucket. |
| **[NFSv3]({{< relref "/protocols/nfs.md" >}})** | **Nada**, por sí solo. `AUTH_UNIX` es una afirmación: el cliente dice «uid 501» y la red no puede contradecirlo. Un bloque `kerberos` levanta esta limitación, y también las [identidades a partir de certificados]({{< relref "/security/nfs-certificates.md" >}}). [TLS]({{< relref "/security/tls.md" >}}) lo cifra, y prueba la máquina, no la persona. |

## La consecuencia, dicha una sola vez {#the-consequence-stated-once}

**Un recurso compartido que nombra quién puede usarlo no se exporta por NFS**, salvo
que Kerberos esté configurado o que NFS tome
[identidades a partir de certificados]({{< relref "/security/nfs-certificates.md" >}}).

No es una advertencia ni una opción: una configuración que dice *photos pertenece a
alice* y un protocolo que entrega photos a quien se conecte no pueden respetarse a la
vez, y ampliar el acceso en silencio es el peor de los dos fallos. El rechazo se
imprime al arrancar y en [`check`]({{< relref "/configuration/check.md" >}}), con el motivo.

## Lo que cada uno necesita de un directorio {#what-each-one-needs-from-a-directory}

Qué protocolos pueden servir a una persona determinada lo decide lo que su origen
puede probar, no solo la configuración. Consulte
[lo que un origen puede probar]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}).

## Lo que está cifrado {#what-is-encrypted}

WebDAV, S3 y NFS admiten `tls = true`; SMB 3 cifra con sus propias claves, y SFTP
es SSH. Consulte [TLS, y certificados de ACME]({{< relref "/security/tls.md" >}}).
