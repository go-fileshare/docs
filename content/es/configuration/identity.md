---
title: "Usuarios, grupos y directorios"
weight: 20
description: "De dónde proceden las personas (bloques user y group, SQL, LDAP, un proveedor de identidad) y qué les permite usar cada origen."
tags: [configuración, identidad, oidc, ldap, sql]
---

Un bloque `user` es todo el directorio de un hogar. Un sitio cuyas personas ya están
en una base de datos o en LDAP no debería copiarlas a un segundo lugar que se quede
desfasado, así que un bloque `users` las lee allí donde están.

```hcl
users "sql" {
  driver   = "postgres"                 # or sqlite, or mysql
  dsn_file = "/etc/fileshare/dsn"       # a DSN holds a password: it lives in a file
  users    = "select login, null, nt_hash, ssh_keys from staff"
  groups   = "select team, member from team_members"
}

users "ldap" {
  url                = "ldaps://ldap.example.org"
  base_dn            = "ou=people,dc=example,dc=org"
  bind_dn            = "cn=reader,dc=example,dc=org"
  bind_password_file = "/etc/fileshare/bind.pw"
}
```

Las **consultas son suyas**, porque las personas de un sitio ya tienen la forma de
ese sitio; un esquema inventado aquí obligaría a copiarlas a un segundo. Las columnas
de la consulta de personas se leen **por posición**, no por nombre: un nombre y luego,
cualquiera de estos, una contraseña, un hash NT, claves SSH (una por línea) y un
secreto TOTP, en ese orden. Un NULL es una credencial que esa persona no tiene, por
eso el ejemplo selecciona `null` para la contraseña. Una columna de contraseña se toma
como la propia contraseña: aquí no hay ningún ajuste para una contraseña con hash.

El lado LDAP lee lo que ya publica un directorio adaptado a Samba: `sambaNTPassword`,
`sshPublicKey`, `memberUid`. Dónde están las personas y los grupos, y qué atributos
los nombran, es configurable: `user_filter` (por defecto
`(objectClass=posixAccount)`), `user_attribute` (`uid`), `group_base_dn`
(`base_dn`), `group_filter` (`(objectClass=posixGroup)`), `group_attribute` (`cn`),
`group_member_attribute` (`memberUid`), y `totp_attribute`, que no tiene valor por
defecto. `sambaNTPassword` y `sshPublicKey` se leen con esos nombres.

{{< callout type="error" >}}
**Ningún bind sin cifrar hacia otra máquina (desde la v0.19.0)**

Un bloque `users "ldap"` rechaza `ldap://` hacia otra máquina salvo que
`start_tls = true`: cada contraseña que comprueba un bind cruzaría la red
sin cifrar. Se acepta `ldaps://`, y también `ldap://` hacia una dirección de
loopback y `ldapi://`, donde no hay nadie en el camino. No existe ningún
interruptor para desactivarlo (go-authn/directory v0.10.0).
{{< /callout >}}

## El orden, y la única excepción {#order-and-the-one-exception}

Los orígenes se consultan **en el orden en que están escritos**, y el primero que
conoce un nombre se lo queda. Los bloques `user` y `group` van primero, así que una
cuenta de servicio anotada localmente no queda sustituida por alguien con el mismo
nombre en LDAP.

**Los grupos son la excepción**: los miembros de un grupo son la **unión** de todos
los orígenes, porque un equipo puede tener personas en un archivo y en una base de
datos.

```hcl
share "photos" {
  allow   = ["@engineers", "alice"]
  writers = ["@owners"]
}
```

{{< callout type="info" >}}
**Expandido al arrancar, y de nuevo en cada recarga**

Desde la v0.10.0, los bloques `users` —SQL, LDAP— se vuelven a leer cada
`reload = "5m"`, con `SIGHUP`, y con el `ReloadDirectory` de la API de
administración; una pertenencia que cambia en LDAP ya no espera a un
reinicio. Lo que una recarga aplica en caliente y lo que reinicia, y por qué
un grupo que se vacía **no** abre su recurso compartido a todo el mundo, se
explica en [volver a leer el directorio]({{< relref "/administration/reload.md" >}}). El
directorio sigue sin consultarse en cada conexión: eso es otro diseño, y no
es este.
{{< /callout >}}

## Lo que un origen puede probar, y lo que necesita cada protocolo {#what-a-source-can-prove-and-what-each-protocol-needs}

Esta es la parte que, si no, un sitio descubre al montar, así que
[`check`]({{< relref "/configuration/check.md" >}}) la dice primero:

| el origen tiene | SMB | S3 | WebDAV | SFTP |
|---|---|---|---|---|
| una contraseña (archivo, o una columna en claro) | sí | sí | sí | — |
| un hash NT (`sambaNTPassword`, `nt_hash`) | sí | **no** | — | — |
| solo un bind (LDAP) | **no** | **no** | sí | — |
| claves públicas, o una CA de confianza | — | — | — | sí |

{{< callout type="error" >}}
**NTLMv2 necesita la contraseña o su MD4, y nada más sirve**

Un cliente nunca envía una contraseña a un servidor SMB —envía una prueba
calculada a partir de ella—, así que un directorio que solo *comprueba*
contraseñas no puede responder a SMB, por buena que sea la comprobación. Es
una propiedad del protocolo, no una limitación de este programa, y ninguna
configuración la cambia. WebDAV solo pregunta «¿es esta la contraseña
correcta?», y eso lo responde un bind.
{{< /callout >}}

Una persona que un directorio nombra pero para la que no prueba nada es legítima —un
listado con los secretos en otro lugar—, y `check` lo indica en una línea en lugar de
dejar que lo descubra.

## Las personas que nombra el proveedor de identidad, no este archivo {#people-the-identity-provider-names-not-this-file}

Una federación —RENATER a través de
[go-authn/bridge](https://github.com/go-authn/bridge), por ejemplo— sabe quién forma
parte de un proyecto, y un recurso compartido puede preguntárselo **a ella** en lugar
de copiar la lista. Con un
[bloque `oidc`]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}):

```hcl
share "photos" {
  image   = "/srv/photos.img"
  allow   = ["oidc:groups:urn:mace:univ-example.fr:photos", "oidc:user:bob@univ-example.fr", "alice"]
  writers = ["oidc:groups:urn:mace:univ-example.fr:photos"]
}
```

`oidc:groups:<value>` es alguien cuyo claim de grupos del token contiene el valor;
`oidc:user:<name>` es alguien a quien nombra el token. La grafía es la de opkssh,
para que un mismo vocabulario diga quién llega a una shell y quién llega a un recurso
compartido. Alguien a quien nombra una regla es conocido por este servidor en la
medida en que lo es por el proveedor —también para los recursos compartidos
abiertos—, y alguien a quien no nombra ninguna regla sigue siendo un desconocido. Las
mismas reglas se aplican por
[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}), donde los
grupos viajan en el certificado en lugar del token.

{{< callout type="error" >}}
**Una regla se refiere solo a las personas del proveedor**

`oidc:user:bob` es el bob que avala el proveedor; una cuenta local llamada
bob, con contraseña, es otra persona y no coincide con ninguna regla. Una
regla sin bloque `oidc`, una mal formada, o una que puede escribir sin tener
permiso para conectarse se **rechaza al arrancar**.
{{< /callout >}}

{{< callout type="error" >}}
**El `alice` del proveedor no es el `alice` local salvo que usted lo diga (desde la v0.20.0)**

Un nombre a secas en `allow` o `writers` es una cuenta local; a un token o a
un certificado del proveedor solo se llega mediante las reglas `oidc:` de un
recurso compartido. Un sitio cuyos nombres en el proveedor SON sus nombres
locales escribe `local_names = true` en el bloque `oidc`, y debería
establecer `domains` junto con ello, o tomar el nombre de un claim que el
proveedor controle: un proveedor en el que las personas eligen su propio
`preferred_username` entregaría si no a cualquiera que se registre los
recursos compartidos de una cuenta local, escritura incluida. Ese era el
comportamiento por defecto antes de la v0.20.0 (auditoría de seguridad F4).
Un nombre federado rechazado por falta de `local_names` se registra con la
línea que hay que añadir, y `fileshare check` indica qué regla está en vigor.

Los certificados de cliente NFS (`identity = "certificate"`) son la
excepción: su CA es la que `client_ca_file` fija precisamente para eso, y
los nombres a secas son la única forma en que un recurso compartido NFS
puede nombrar a alguien, así que sus nombres son nombres locales diga lo que
diga `local_names`.
{{< /callout >}}

### Qué instituciones {#which-institutions}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
  domains  = ["univ-a.fr", "univ-b.fr"]   # nobody else from the federation gets in
}

share "projet-x" {
  image   = "/srv/projet-x.img"
  allow   = ["oidc:groups:urn:mace:univ-a.fr:projet-x", "oidc:domain:univ-b.fr"]
  writers = ["oidc:groups:urn:mace:univ-a.fr:projet-x"]
}
```

`domains` se comprueba en la autenticación, tanto por SFTP como por WebDAV, y para
los certificados de cliente NFS: un nombre debe ser `<algo>@<uno de ellos>`, comparado
entero (`evilunivb.fr` no es `univb.fr`). `oidc:domain:` es la misma prueba para un
solo recurso compartido. Se puede confiar en el dominio tanto como en el proveedor:
go-authn/bridge descarta un eppn o un subject-id cuyo ámbito no le conceden los
metadatos de federación del IdP.

**Los grupos** son lo que publica el IdP de la institución, convertido en el claim
`groups` por el `claims { groups = [...] }` de go-authn/bridge: `eduPersonEntitlement`
por defecto (los grupos de un laboratorio o de una VO), o
`eduPersonScopedAffiliation` (`staff@univ-a.fr`, `student@univ-b.fr`).
