---
title: "check, antes de reiniciar algo que la gente está usando"
linkTitle: "check, antes de reiniciar"
weight: 30
description: "Lo que fileshare check imprime sobre una configuración antes de servirla, y cómo leerlo."
tags: [configuración, check]
---

```
$ fileshare check /etc/fileshare.d
ATTIC

SHARE    IMAGE             FILESYSTEM  WHO MAY CONNECT           WHO MAY WRITE             SMB  WEBDAV  NFS  SFTP
photos   /srv/photos.img   fat32       alice and bob             alice                     yes  yes     NO   yes
scratch  /srv/scratch.img  ext4        anyone who authenticates  anyone who authenticates  yes  yes     yes  yes

photos is not served over nfs: it is restricted to alice and bob, and NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client makes about itself and the wire cannot disagree with it. A kerberos block lifts this: sec=krb5 carries a principal a ticket proves; so does identity = "certificate" on the nfs serve block: RPC-over-TLS with a client certificate naming the person

USER   FROM                      AUTHENTICATES WITH                         SMB  WEBDAV  SFTP
alice  the configuration file    a password and 1 key from /etc/…/alice.pw  yes  yes     yes
bob    the configuration file    a password from /etc/…/bob.pw              yes  yes     -
dora   ldaps://ldap.example.org  a password check and an NT hash            yes  yes     -
eli    ldaps://ldap.example.org  a password check                           -    yes     -

this configuration can be served
```

Cada imagen se abre como la abre el servidor y se vuelve a cerrar; no se sirve nada.
Eso es **en lectura y escritura** para un recurso compartido que no indica
`read_only = true`; solo un dispositivo se abre siempre en solo lectura, y en
exclusiva, de modo que un dispositivo que tiene el servidor en ejecución puede
rechazarse aquí por estar ocupado.

Imprime **de dónde procede una credencial y nunca lo que contiene**, y las columnas
por persona son la razón de ser de
[`Identity.Can`](https://github.com/go-authn/directory): eli está en el mismo grupo
que dora y SMB sigue sin poder servirle, porque LDAP guarda su contraseña y no se la
da a nadie.

## Cómo leer las columnas {#reading-the-columns}

| | |
|---|---|
| un sistema de archivos con un asterisco | el controlador lo **afirmó** el recurso compartido, no se reconoció; consulte [recursos compartidos]({{< relref "/configuration/shares.md" >}}) |
| `NO` bajo un protocolo | ese recurso compartido no se exporta ahí, y el motivo se imprime debajo de la tabla |
| `-` bajo un protocolo, por recurso compartido | el `protocols` del recurso compartido deja ese fuera |
| `-` bajo un protocolo, por usuario | el origen de esa persona no puede probar lo que el protocolo necesita; consulte [identidad]({{< relref "/configuration/identity.md" >}}) |

## Qué más dice {#what-else-it-says}

Después de las tablas, `check` indica lo que una persona necesita saber antes de
confiar en la configuración, cuando procede:

- con un bloque `oidc`, que sus tokens solo se aceptan por WebDAV, y si los nombres
  del proveedor son nombres locales (`local_names`); consulte
  [identidad]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}});
- por protocolo, lo que está **cifrado** y lo que va **sin cifrar a propósito**;
  consulte [TLS]({{< relref "/security/tls.md#what-check-says" >}});
- de dónde procede una **lista de revocación**, y qué ocurre mientras no se puede
  descargar; consulte [qué revoca qué]({{< relref "/security/_index.md" >}});
- qué recursos compartidos se **retiraron de servicio** mediante el `DisableShare` de
  la API de administración, y no se sirven; consulte
  [la API de administración]({{< relref "/administration/_index.md#disabling-a-share" >}}).

Para un servidor con una KRL, identidades NFS a partir de certificados y señales
compartidas, esa parte dice:

```
sftp: the provider's SSH certificates are checked against the KRL at https://bridge.example.org/ssh/krl, at login and on every operation; while it is unknown or older than 1h0m0s they are refused

federated people: revocations come from the shared signals transmitter https://bridge.example.org (CAEP session-revoked, polled); whatever the provider issued them before a revocation is refused -- tokens, SSH and OpenPubkey certificates, NFS certificates -- and open sessions stop. Refused while it is not heard from within 10m0s; revocations kept 192h0m0s in /var/lib/fileshare/revocations.json

nfs: identities come from client certificates signed by /etc/fileshare/bridge-x509-ca.pem, revoked by the CRL at https://bridge.example.org/x509/crl (refused while it is unknown or older than 1h0m0s).
     ⛔ a Linux client's certificate belongs to a MOUNT: every user of that mount is the person it names
```

`check` no descarga las listas: indica de dónde proceden y qué provocará su
ausencia. En cambio, el documento de descubrimiento y las claves del proveedor de
identidad **sí** se leen: `check` rechaza un bloque `oidc` cuyo emisor no se puede
alcanzar.
