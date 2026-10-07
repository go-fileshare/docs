---
title: "Volver a leer el directorio"
weight: 30
description: "Cómo y cuándo vuelve a leer el servidor sus directorios de usuarios, y lo que una recarga hace a los recursos compartidos y a las conexiones abiertas."
tags: [administración, identidad, recarga]
---

```hcl
reload = "5m"   # and on SIGHUP, and on the admin API's ReloadDirectory
```

Las personas proceden de directorios que otras cosas modifican: go-authn/bridge
escribe una contraseña de aplicación en una tabla, y borra la fila cuando la persona
se desactiva. Sin recarga, el servidor sirve a quien estuviera cuando arrancó. Desde
la v0.10.0 las vuelve a leer:

- en cada intervalo `reload`, una duración de Go de **al menos un segundo**: un
  directorio leído muchas veces por segundo es una carga para la base de datos de
  alguien, no un servidor más actualizado;
- con **`SIGHUP`** (Unix);
- con el **`ReloadDirectory`** de la [API de administración]({{< relref "/administration/_index.md" >}}),
  que responde con quién se añadió, se eliminó y cambió, si empezó una nueva
  generación, y notas que una persona debería leer: un grupo que ya no existe,
  alguien a quien nombra un recurso compartido y que ya no está en el directorio.

Sin `reload`, el directorio solo se lee al arrancar, con `SIGHUP` y con
`ReloadDirectory`.

Lo que se vuelve a leer son los bloques `users` —SQL, LDAP—. Los bloques `user` y
`group` del archivo de configuración **son** la configuración, leída al arrancar.

## Lo que hace una recarga depende de lo que cambió {#what-a-reload-does-depends-on-what-changed}

La línea se traza en la **revocación**:

| lo que cambió | lo que ocurre |
|---|---|
| **solo altas**: alguien nuevo cuya llegada no cambia las listas expandidas de ningún recurso compartido (una nueva contraseña de aplicación, una persona a la que los recursos compartidos llegan mediante reglas `oidc:` o de ninguna forma) | se añade **en caliente**, incluido el servidor SMB en ejecución, y **no se toca ninguna conexión**. Un bridge que crea contraseñas de aplicación todo el día no molesta a nadie. |
| **cualquier retirada**: alguien que se ha ido, una credencial cambiada, un recurso compartido cuyas listas expandidas cambiaron | una **nueva generación**, y las conexiones de la anterior **se cierran**, exactamente igual que con un [cambio de administración]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}}). Una sesión que sobrevive a la baja de su persona es justo lo que una recarga existe para terminar. |
| **un directorio que no se puede leer**: la base de datos inalcanzable, una consulta que falla | **nada cambia**. Una caída no debe vaciar un servidor de archivos, y «nadie» es el aspecto que tiene una lectura fallida. |

Alguien nuevo que se une a un grupo que nombra un recurso compartido *sí* cambia ese
recurso compartido, y pasa por una nueva generación como cualquier otro cambio en él:
SMB fija las listas de un recurso compartido cuando este arranca.

Un recurso compartido que nombra a alguien que se ha ido, o a un grupo que ya no
existe, se sirve **sin ellos** —la dirección segura— y se indica, en lugar de
rechazarse como lo rechaza un arranque: rechazarlo conservaría las listas antiguas, y
las listas antiguas son el acceso que se acaba de retirar.

## Un grupo vacío no es todo el mundo {#an-empty-group-is-not-everyone}

{{< callout type="error" >}}
**Un recurso compartido escrito para `@engineers` cuyo último ingeniero se ha ido no se sirve a nadie**

Un `allow` vacío significa «cualquiera que se autentique». Así que lo que
decide si un recurso compartido está abierto es lo que se **escribió**, no
aquello en lo que se expande ahora: un recurso compartido escrito para un
grupo sigue cerrado cuando ese grupo se vacía. Un recurso compartido así no
se ofrece en absoluto por SMB, cuyo `AllowUsers` vacío se interpretaría como
«todo el mundo».
{{< /callout >}}

## Vigilarlo {#watching-it}

`fileshare_directory_reloads_total{result}` cuenta las recargas que no encontraron
nada (`unchanged`), que añadieron en caliente (`added`), que iniciaron una generación
(`swapped`) o que no pudieron leer el directorio (`failed`);
`fileshare_directory_people` indica cuántas personas contenía la última lectura.
Consulte [salud y métricas]({{< relref "/administration/health.md" >}}).

{{< callout type="info" >}}
**Lo que no alcanza una recarga**

Las personas que el proveedor de identidad avala mediante un token o un
certificado no están en el directorio, así que ninguna recarga les
concierne. Retirarles el acceso a **ellas** es tarea de las
[listas de revocación y las señales compartidas]({{< relref "/security/_index.md" >}}).
{{< /callout >}}
