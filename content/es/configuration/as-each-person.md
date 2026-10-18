---
title: "Directorios servidos con la identidad de cada persona"
weight: 15
description: "identity = \"system\": un recurso de directorio cuyos archivos abre un worker que se ejecuta como la persona conectada; el núcleo aplica los permisos del árbol, y lo que crea le pertenece."
tags: [configuración, recursos, identidad, seguridad]
---

Normalmente un único proceso fileshare sirve a todos con una sola cuenta:
fileshare decide quién puede hacer qué, y cada archivo que crea pertenece a
esa cuenta. Un recurso de directorio puede servirse en cambio **con la
identidad de cada persona**. Los archivos de quien se conectó los abre un
proceso con su uid, su gid y sus grupos: el núcleo aplica los permisos del
árbol, y lo que crea le pertenece.

```hcl
identity {
  service_user = "fileshare"   # la cuenta con la que se sirven los protocolos
  min_uid      = 1000          # el valor por defecto; el uid 0 se rechaza siempre
  min_gid      = 1000
  deny_uids    = [65534]
  idle         = "2m"          # cuánto tiempo se conserva un worker sin uso
}

share "home" {
  directory = "/srv/home"
  identity  = "system"
}
```

## Tres procesos {#three-processes}

`fileshare serve` debe **arrancarse como root**. Sigue siendo root solo para
arrancar el worker de cada persona. Todo lo que da a la red se ejecuta en un
segundo proceso, como `service_user`.

| proceso | se ejecuta como | hace |
|---|---|---|
| el **lanzador** (`fileshare serve`) | root | abre los puertos, arranca el front, arranca workers cuando el front lo pide. No habla ningún protocolo. |
| el **front** | `service_user`, con los grupos de esa cuenta | habla los protocolos y autentica, como `serve` siempre ha hecho. Para un recurso así pide al lanzador una **persona y un recurso**, nunca un uid ni una ruta. |
| un **worker** | la persona | sirve un recurso al front, por un socket que el lanzador le entregó |

Un worker comprueba primero que se ejecuta exactamente con el uid, el gid y
los grupos esperados, y que `setuid(0)` falla. Solo entonces abre el árbol,
mediante un `os.Root`. Un mismo worker sirve todas las conexiones de esa
persona a ese recurso, por SFTP y por WebDAV, y se detiene cuando nada lo ha
usado durante `idle`. Cuando el servidor se detiene, los workers se detienen
con él.

## Quién es una persona en este sistema {#who-a-person-is-on-this-system}

El lanzador resuelve a la persona **por sí mismo**, en este orden:

1. **El directorio**, si lleva `uidNumber` y `gidNumber`:
   - los `attributes = { uidNumber = "2001", gidNumber = "2001" }` de un bloque `user`;
   - una fuente LDAP con `attributes = ["uidNumber", "gidNumber"]`;
   - la consulta `users_attributes` de una fuente SQL.

   Los grupos suplementarios son los grupos que listan a la persona (los
   `members` de un bloque `group`, `memberUid`) y los que la persona nombra
   (`memberOf`), cada uno por su `gidNumber`. Un grupo sin él no es un grupo
   del sistema.
2. **`/etc/passwd` y `/etc/group`**, por nombre.

**Rechazados**: el uid 0, el gid 0, `(uid_t)-1`, un uid por debajo de
`min_uid`, un gid primario por debajo de `min_gid`, y todo uid de
`deny_uids`. Un grupo suplementario por debajo de `min_gid` se omite, y el
registro lo dice. Una persona que no aparece en ninguno de los dos se rechaza
también: no hay cuenta compartida de reserva. Una persona avalada por el
proveedor de identidad obtiene un worker solo con
[`local_names`]({{< relref "/configuration/identity.md" >}}), que hace de su
nombre un nombre local.

Así, un front comprometido solo puede pedir una persona que existe y está
permitida, en un recurso que existe. Sigue siendo él quien decide **quién**
se ha autenticado, y el lanzador no puede comprobarlo por él.

## Lo que ve un cliente {#what-a-client-sees}

| | SFTP | WebDAV |
|---|---|---|
| se toma un worker | al conectarse, para toda la conexión | por petición; los bloqueos viven lo que vive el worker |
| el lanzador rechaza a la persona | el recurso no está en su árbol (un árbol vacío es un inicio de sesión rechazado) | `403` |
| no puede arrancar ningún worker | igual | `503` |

El motivo va al registro del servidor, nunca al cliente.

{{< callout type="warning" >}}
**Todavía no por SMB, NFS ni S3**

Rechazan un recurso así, y [`check`]({{< relref "/configuration/check.md" >}})
lo dice. SMB y NFS necesitan en sus bibliotecas un sistema de archivos por
sesión. NFS solo lo hará bajo Kerberos, porque `AUTH_UNIX` es la mera
afirmación del cliente (RFC 5531 §14).
{{< /callout >}}

## Lo que pide al sistema {#what-it-asks-of-the-system}

- **Unix**: Linux, los BSD y macOS. En Windows el recurso se rechaza.
- **Legibles por `service_user`**: la configuración y cada archivo que nombra
  (archivos de contraseñas, clave de host, certificados), porque los lee el
  front.
- **Ejecutable por las personas**: el binario, porque cada worker lo ejecuta
  como su persona.
- **Un reinicio** para añadir o cambiar un recurso así. El lanzador lee la
  configuración una vez, y rechaza un recurso que no conoce en lugar de dejar
  que el front lo sirva.
- `--isolate` no va con ello.

El diseño, y el porqué de cada decisión, están en
[docs/identity.md](https://github.com/go-fileshare/fileshare/blob/main/docs/identity.md).
