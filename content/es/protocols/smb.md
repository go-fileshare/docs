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

## Kerberos: un ticket en lugar de una contraseña {#kerberos}

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"   # nfs/fs.example.org y cifs/fs.example.org
}

serve "smb" {
  addr = "0.0.0.0:445"
  ntlm = false          # solo Kerberos: nunca se comprueba una contraseña
}
```

El bloque `kerberos` que permite a NFS distinguir a las personas también permite a SMB
aceptar un ticket. **Un solo keytab** guarda ambos servicios y cada protocolo responde
por el suyo: SMB solo lee los principales `cifs/<host>` del realm del bloque, así que
un ticket emitido para `nfs/` se rechaza aquí. Un keytab sin principal `cifs/` deja
SMB como estaba, solo NTLM, y el servidor lo dice al arrancar.

El principal pasa por la misma regla que en NFS (la `alice` de otro realm no es nadie)
y después debe ser **alguien a quien esta configuración conoce**: un ticket prueba que
el KDC conoce a la persona, no que este servidor la conozca. Las listas `allow` y
`writers` del recurso se aplican como con una contraseña. Una sesión termina con su
ticket, y un cliente Windows se autentica de nuevo por sí solo.

Desde fileshare v0.50.0, el **keytab** debe ser un archivo regular, propiedad de root
o de la cuenta con la que corre fileshare, que no sea legible ni escribible por
otros ni escribible por su grupo: `root:fileshare` `0640` y `fileshare:fileshare`
`0600` se aceptan, `krb5:fileshare` `0640` se rechaza. Vale para SMB y NFS. El nombre
y el realm del cliente salen del **ticket**, nunca del autenticador que el cliente
también envía (jcmturner/gokrb5#577, RFC 4120 sección 3.2.3); NFS necesita
go-authn/krb5 v0.3.4 o posterior, que incluye v0.50.0.

`ntlm = false` se rechaza al cargar sin un bloque `kerberos`, y el servidor se niega a
arrancar si el keytab no contiene ninguna clave `cifs/` para el realm: un servidor en
el que nadie puede entrar no es un servidor. Con el bloque, una persona cuyo
directorio no guarda ningún hash NT puede usar SMB igualmente, y
[`check`]({{< relref "/configuration/check.md" >}}) lo indica.

Un equipo Windows **fuera de un dominio** llega con un ticket de este realm: una vez
`ksetup /addkdc` y `ksetup /addhosttorealmmap`, un ticket de concesión de tickets en
la sesión de inicio y `net use \\fs.example.org\photos`, por nombre, porque con una
dirección Windows recurre a NTLM. Medido con Windows 11 25H2 (sin unir): el cliente
ofrece MS-KRB5, Kerberos, NEGOEX y NTLMSSP, envía un AP-REQ de Kerberos, monta, lista,
lee y escribe con la firma exigida, y no envía ningún mensaje NTLM.

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

## Solicitudes en vuelo {#requests-in-flight}

Desde go-filesystems/smb v0.8.1, una conexión mantiene hasta 32 solicitudes en vuelo, y el servidor hasta 128 entre todas las conexiones. Las respuestas se asocian por `MessageId`. Una trama que contiene un único `READ` de un archivo normal se ejecuta junto a otras; las cadenas compuestas (respondidas enteras y en orden), las solicitudes asíncronas y `WRITE` se ejecutan solas. Una trama dispone de 30 s para llegar una vez iniciada. El mecanismo es el módulo compartido go-filesystems/inflight, que también usan SFTP y NFS.
