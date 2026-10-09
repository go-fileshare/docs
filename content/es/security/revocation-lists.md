---
title: "Revocar certificados SSH: la KRL"
weight: 20
description: "Revocar los certificados SSH del proveedor de identidad con una KRL de OpenSSH firmada y fechada, comprobada al iniciar sesión y en cada operación SFTP."
tags: [seguridad, revocación, sftp, ssh]
---

Un certificado firmado por la CA SSH del proveedor de identidad —consulte
[SFTP para las personas que el proveedor avala]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})—
se comprueba al iniciar sesión. Desactivar a alguien en go-authn/bridge revoca sus
tokens y elimina sus contraseñas de aplicación, cosa que aquí aplica una
[recarga del directorio]({{< relref "/administration/reload.md" >}}); pero sin una lista, un
certificado ya emitido seguiría abriendo SFTP hasta caducar, y una sesión SFTP ya
abierta seguiría abierta.

Desde la v0.12.0, una lista lo impide:

```hcl
oidc {
  issuer          = "https://login.example.org"
  audience        = "fileshare"
  ssh_ca_file     = "/etc/fileshare/bridge-ca.pub"
  ssh_krl_url        = "https://bridge.example.org/ssh/krl"   # or ssh_krl_file
  ssh_krl_state_file = "/var/lib/fileshare/ssh.krl.state"    # the order, across a restart
  ssh_krl_max_age    = "1h"                                  # default; ssh_krl_refresh = "1m"
}
```

go-authn/bridge publica los certificados SSH que ha revocado —una persona
desactivada, un IdP desactivado— como una **KRL** de OpenSSH (`PROTOCOL.krl`, la
lista que lee el `RevokedKeys` de `sshd`), y fileshare guarda una copia, leída con
[go-authn/krl](https://github.com/go-authn/krl).

## Rechazado al iniciar sesión, y en mitad de una sesión {#refused-at-login-and-in-the-middle-of-a-session}

Un certificado revocado se rechaza al iniciar sesión, y **una sesión que ya abrió
deja de recibir servicio**. SSH comprueba un certificado una sola vez, durante el
handshake; por eso cada operación de una sesión SFTP federada vuelve a consultar la
KRL, **incluidos los archivos abiertos**. La primera operación después de que cambie
la copia de la lista de este servidor se rechaza.

## Campos {#fields}

| campo | por defecto | |
|---|---|---|
| `ssh_krl_url` | | dónde se descarga la lista, **solo HTTPS** |
| `ssh_krl_file` | | o un archivo, por ruta absoluta; uno u otro, nunca ambos |
| `ssh_krl_ca_file` | las del sistema | fija las autoridades frente a las que se comprueba el servidor HTTPS de la lista |
| `ssh_krl_refresh` | `1m` | con qué frecuencia se descarga, al menos un segundo |
| `ssh_krl_max_age` | `1h` | qué antigüedad puede tener la última copia válida antes de que se rechacen los certificados que gobierna |
| `ssh_krl_state_file` | | dónde se guarda la última lista verificada, para que un reinicio recuerde el orden |

Un `max_age` más corto que `refresh` se rechaza: cada copia estaría obsoleta antes
de la siguiente descarga. Una KRL sin `ssh_ca_file` también se rechaza: revoca los
certificados del proveedor, y sin esa CA no hay ninguno.

## Falla en modo cerrado {#it-fails-closed}

{{< callout type="error" >}}
**Mientras no se pueda descargar la KRL, los certificados del proveedor se rechazan**

Mientras no se pueda descargar la lista, o su última copia válida sea más
antigua que `ssh_krl_max_age`, se rechazan **todos** los certificados
firmados por la CA del proveedor: una comprobación de revocación que deja
pasar todo cuando la lista es inalcanzable es la que derrota un atacante
capaz de bloquear la lista. La lista debe servirse entonces con la misma
fiabilidad que los inicios de sesión que gobierna.
`fileshare_revocation_list_age_seconds{list="ssh_krl"}` es la métrica sobre
la que alertar antes de alcanzar `max_age`; consulte
[salud y métricas]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

## Firmada y fechada, desde `ssh_krl_url` {#signed-and-dated-from-ssh_krl_url}

Desde la v0.15.0, **una KRL procedente de `ssh_krl_url` debe estar firmada e indicar
cuándo caduca**, tal como describe
[go-authn/revocation](https://github.com/go-authn/revocation) y como la sirve
go-authn/bridge a partir de la v0.10.0:

- **La firma:** `<url>.sig` es una firma SSHSIG separada de la CA SSH del proveedor
  (`ssh_ca_file`, que por tanto `ssh_krl_url` exige), en el espacio de nombres
  `krl@go-authn.github.io`. Se descarga con `If-Match` sobre el ETag de la lista, y
  se vuelve a descargar de inmediato si la lista se ha vuelto a emitir entretanto.
- **La caducidad:** la extensión `expires@go-authn.github.io`. Pasada esa fecha, la
  lista no está vigente, por reciente que sea el `304` que la confirmó.

HTTPS (`ssh_krl_ca_file` fija sus autoridades) autentica al servidor que respondió,
no a la lista: un espejo, una caché o un servidor web comprometido podrían servir una
vacía, y nada en una KRL por sí sola indica que esté obsoleta. Una lista que no se
verifica se rechaza, y se conserva la última copia válida. Una lista colocada a mano
con `ssh_krl_file` merece la misma confianza que el sistema de archivos que la
contiene, y no necesita ni firma ni caducidad; si lleva alguna, se respeta.

{{< callout type="warning" >}}
**Actualizar a la v0.15.0**

Un `ssh_krl_url` que sirve una KRL sin firmar, o sin caducidad, ya no se
acepta: los certificados del proveedor se rechazan entonces, como con
cualquier lista que no se puede obtener. Ejecute primero go-authn/bridge
v0.10.0 o posterior.
{{< /callout >}}

## Nunca hacia atrás, tampoco tras un reinicio {#never-backwards-across-a-restart-too}

Desde la v0.16.0, una lista se rechaza si es más antigua que la que se tiene: una
versión inferior (la versión de la KRL, el número de una CRL), o la misma versión
emitida antes. Con `ssh_krl_state_file` (`crl_state_file` para
[NFS]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})), la última lista
verificada se guarda en disco, se vuelve a verificar al arrancar y ordena la
siguiente. Por sí sola no cuenta como vigente hasta que se descarga una lista. Sin
ella, un reinicio olvida el orden, y fileshare lo indica al arrancar: se aceptaría
una lista más antigua, aún firmada y sin caducar.

Estos archivos, como `ssh_ca_file`, los archivos de CA, los archivos de lista y
(desde la v0.17.0) `authorized_keys_file`, `dsn_file`, `bind_password_file` y el
`ca_file` de `ssf`, **no pueden estar dentro de un recurso compartido**: quien
escribiera allí decidiría quién entra.

## Un `sshd` normal junto a fileshare {#a-plain-sshd-next-to-fileshare}

La misma verificación está disponible sin fileshare: el `authn-revokd` de
go-authn/revocation descarga las listas, conserva solo las que se verifican, guarda
su orden en disco, escribe `RevokedKeys` para `sshd` y, cuando una lista caduca,
escribe una que revoca la propia CA, de modo que `sshd` también falla en modo
cerrado.

## Lo que no cubre {#what-it-does-not-cover}

**Los certificados OpenPubkey (opkssh) no están en ninguna KRL**: no los emitió nada
más que la propia clave de la persona. Los limita `opkssh_max_age`, y los retiran
las [señales compartidas]({{< relref "/security/shared-signals.md" >}}). Lo mismo ocurre con un token de acceso por WebDAV.

El mismo mecanismo —descargado, conservado, en modo cerrado ante el fallo— transporta
la **CRL X.509** de las
[identidades NFS a partir de certificados]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}).
