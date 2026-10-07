---
title: "La configuración"
linkTitle: "Configuración"
weight: 10
description: "Cómo se organiza la configuración HCL, qué contiene un bloque user y qué se rechaza al arrancar."
tags: [configuración]
---

Un archivo, o un directorio de archivos pequeños. `--config /etc/fileshare.d` los
fusiona, de modo que un usuario en un archivo y un recurso compartido en otro forman
un conjunto.

```hcl
name = "ATTIC"

user "alice" { password_file = "/etc/fileshare/alice.pw" }
user "bob"   { password_file = "/etc/fileshare/bob.pw" }

group "family" { members = ["alice", "bob"] }

share "photos" {
  image   = "/srv/photos.img"
  allow   = ["@family"]        # a group, or a person, in either list
  writers = ["alice"]          # bob gets it read-only
}

share "scratch" {
  image = "/srv/scratch.img"   # anyone who authenticates, read-write
}

tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true                  # HTTP Basic is the password: never in the clear
}
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

{{< callout type="error" >}}
**Desde la v0.9.0, WebDAV con contraseñas no se sirve sin cifrar**

`serve "webdav" { addr = "0.0.0.0:8080" }` —el ejemplo que mostraba esta
página hasta entonces— ahora se **rechaza al arrancar** en cuanto alguien se
autentica: HTTP Basic es la contraseña en cada petición. Indique
`tls = true` con un bloque `tls`, como arriba, o `plaintext = true` cuando
un proxy termina TLS delante. Consulte [TLS]({{< relref "/security/tls.md" >}}).
{{< /callout >}}

Un recurso compartido también puede ser un [dispositivo]({{< relref "/configuration/shares.md#a-device-not-only-an-image" >}})
o un [directorio del host]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}) en lugar
de una imagen.

## Lo que puede contener un bloque `user` {#what-a-user-block-can-carry}

Todos los ejemplos de arriba usan `password_file`, que es el caso habitual y fue
durante un tiempo el único que leía este programa. No es todo el bloque, y la
diferencia decide **por qué protocolos se puede servir a una persona**; por eso
conviene tenerlo en un solo lugar en vez de descubrirlo al montar.

| campo | qué es | qué aporta |
|---|---|---|
| `password` | una contraseña, en el archivo | todos los protocolos a los que responde una contraseña |
| `password_file` | lo mismo, en un archivo propio | lo mismo, sin el secreto en la configuración |
| `nt_hash` | `MD4(UTF16LE(password))`, 32 caracteres hexadecimales | **SMB**, para una persona cuya contraseña este sitio no guarda |
| `authorized_keys` | líneas `authorized_keys`, en línea | **SFTP** |
| `authorized_keys_file` | lo mismo, desde un archivo | **SFTP** |
| `totp_secret` | un secreto de código de un solo uso en base32 | nada todavía: se lee y se comprueba, y ningún protocolo de aquí pide un segundo factor |

⛔ `password` y `password_file` juntos se rechazan al arrancar, nombrando a la
persona: dos respuestas a «cuál es su contraseña» plantean la pregunta de cuál gana,
y una configuración no debería tener que leerse dos veces para averiguarlo. Lo mismo
ocurre con `authorized_keys` y `authorized_keys_file` juntos.

⛔ Un bloque `user` sin `password`, `password_file`, `authorized_keys` ni
`authorized_keys_file` también se rechaza —*has no way to authenticate*—, salvo que
`trusted_user_ca_file` esté establecido. `nt_hash` y `totp_secret` no cuentan aquí.

{{< callout type="default" >}}
**Un usuario en línea SÍ puede servirse por SMB**

Hasta hace poco, este bloque solo leía los campos de contraseña, así que
servir SMB a alguien anotado aquí significaba dar a este archivo su
contraseña en claro, o trasladarlo a una base de datos. Con `nt_hash` ya no:
un sitio que guarda lo que guarda Samba puede escribir eso en su lugar. El
bloque sigue necesitando uno de los campos de arriba a su lado —claves, por
ejemplo— o `trusted_user_ca_file`: `nt_hash` solo se rechaza al arrancar.
Consulte
[lo que un origen puede probar]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}),
que es la misma tabla un nivel más arriba.
{{< /callout >}}

Un grupo se escribe `@nombre` allí donde podría ir una persona.

## Lo que se rechaza al arrancar, y no más tarde {#what-is-refused-at-startup-rather-than-later}

{{< callout type="warning" >}}
**Un nombre que no pertenece a nadie**

`allow = ["alise"]` dejaría si no a Alice fuera de su propio recurso
compartido y arrancaría tan tranquilo. Un nombre que no conoce ningún bloque
`user` ni ningún [directorio `users`]({{< relref "/configuration/identity.md" >}}) se
rechaza, igual que un `@grupo` que ninguno de ellos tiene.
{{< /callout >}}

{{< callout type="warning" >}}
**Un recurso compartido que nombra quién puede usarlo, por NFS**

Una configuración que dice *photos pertenece a alice* y un protocolo que
entrega photos a quien se conecte no pueden respetarse a la vez. Un recurso
compartido así no se exporta por NFS —salvo que un bloque `kerberos` o
`identity = "certificate"` permita a NFS distinguir a las personas—, y un
bloque serve `nfs` que se queda sin nada que servir se rechaza. Consulte
[NFS]({{< relref "/protocols/nfs.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Un proveedor sin dónde poner su palabra**

Un bloque [`oidc`]({{< relref "/protocols/webdav.md" >}}) se rechaza salvo que se sirva
WebDAV —el único protocolo que transporta una cabecera `Authorization`— o que
se sirva SFTP con los certificados del proveedor activados (`ssh_ca_file` u
`opkssh_client_id`, consulte [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})).
SMB y NFS no tienen dónde poner ni lo uno ni lo otro.
{{< /callout >}}

{{< callout type="warning" >}}
**Contraseñas sin cifrar**

WebDAV en una dirección que otras máquinas pueden alcanzar, sin `tls = true`
ni `plaintext = true`, mientras haya alguien que se autentique. Consulte
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Un bloque que no se puede servir de forma segura**

Un bloque `admin` sin `state_file`, o que escucha en TCP sin TLS mutuo; un
bloque `tls` que ningún bloque `serve` usa; un `reload` de menos de un
segundo; una lista de revocación descargada por HTTP plano. Cada caso se
describe donde corresponde: [administración]({{< relref "/administration/_index.md" >}}),
[TLS]({{< relref "/security/tls.md" >}}), [recarga]({{< relref "/administration/reload.md" >}}),
[revocación]({{< relref "/security/_index.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Un protocolo con el que este binario no se compiló**

Se le dice *eso*, y no «no existe tal protocolo»: la diferencia entre una
errata y una [etiqueta de compilación]({{< relref "/operations/build-tags.md" >}}).
{{< /callout >}}

Un bloque `serve` sin `addr` escucha en loopback, `127.0.0.1`, en un puerto que no
requiere privilegios: 4445 para SMB, 8080 para WebDAV, 2222 para SFTP, 2049 para
NFS, 9000 para S3.

## `protocols` en un recurso compartido {#protocols-on-a-share}

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

Indica qué protocolos lo sirven. Útil por sí mismo —un recurso compartido declarado
solo SMB es un recurso compartido del que nunca se informa al proceso WebDAV— y
obligatorio en algunas configuraciones con [`--isolate`]({{< relref "/operations/isolation.md" >}}).
Un bloque `serve` que acabaría sin servir nada también se rechaza, antes de abrir
ninguna imagen, nombrando los recursos compartidos que se le retiraron y por qué,
salvo que haya un bloque `admin` para darle recursos compartidos más adelante.
