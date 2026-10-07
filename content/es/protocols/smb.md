---
title: "SMB — NTLMv2"
linkTitle: "SMB: NTLMv2"
weight: 10
description: "SMB con NTLMv2: lo que debe guardar un directorio, el puerto privilegiado con aislamiento, y lo que una recarga del directorio hace a las conexiones abiertas."
tags: [protocolos, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

La contraseña **nunca cruza la red**: el cliente envía una prueba calculada a partir
de ella. El recurso compartido indica a quien solo puede leer que es así —en la
máscara de acceso, antes de que lo intente—, en lugar de hacer fallar una escritura
más tarde.

## Lo que debe guardar un directorio {#what-a-directory-must-hold}

NTLMv2 necesita la propia contraseña o su MD4 (`sambaNTPassword`, una columna
`nt_hash`). **Nada más sirve.**

Un directorio que solo *comprueba* contraseñas —un bind LDAP, una columna bcrypt—
no puede responder a SMB, por buena que sea la comprobación, porque el servidor tiene
que calcular la misma prueba que calculó el cliente. Es una propiedad del protocolo,
no una limitación de este programa.

[`check`]({{< relref "/configuration/check.md" >}}) imprime un `-` en la columna SMB para esa
persona, en el momento de la configuración, en lugar de dejar que lo descubra al
montar.

## Un puerto privilegiado sin un servidor privilegiado {#a-privileged-port-without-a-privileged-server}

Con [`--isolate`]({{< relref "/operations/isolation.md" >}}) en Unix, el proceso padre se vincula
al 445 y pasa la escucha al hijo, de modo que el proceso que realmente habla SMB
nunca necesita el privilegio. Windows no tiene `ExtraFiles`, así que allí el hijo se
vincula él mismo a la dirección; el aislamiento es el mismo, la parte del puerto
privilegiado no.

## Verificado con un cliente que este proyecto no escribió {#verified-against-a-client-this-project-did-not-write}

[go-smb2](https://github.com/cloudsoda/go-smb2) realiza veinte lecturas
concurrentes por SMB mientras otras veinte pasan por WebDAV, con `-race`, en la CI.
macOS monta un recurso compartido por SMB mientras `curl` escribe en la misma imagen
por WebDAV, y vuelve a leer lo que WebDAV escribió.

## Cuando cambian las personas {#when-the-people-change}

SMB fija quién puede conectarse a un recurso compartido **cuando se añade el recurso
compartido**, y lo comprueba una vez por conexión de árbol (tree connect). Así, a
alguien nuevo que encuentra una [recarga del directorio]({{< relref "/administration/reload.md" >}})
se le añade en caliente, incluido el servidor SMB en ejecución, sin tocar ninguna
conexión; pero cualquier cosa que se retire —una persona, una credencial, las listas
expandidas de un recurso compartido— es una nueva generación, y las conexiones SMB
abiertas se cierran, de modo que una revocación alcanza también a las sesiones ya
abiertas. Un recurso compartido cuyo grupo `allow` se ha vaciado **no se ofrece por
SMB en absoluto**: su `AllowUsers` vacío se interpretaría como «todo el mundo».
