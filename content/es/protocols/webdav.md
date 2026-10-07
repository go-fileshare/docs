---
title: "WebDAV — Basic, o un token bearer"
linkTitle: "WebDAV: Basic, o un token bearer"
weight: 20
description: "WebDAV con HTTP Basic o un token bearer OIDC, el TLS que exige, y cómo un token se convierte aquí en una persona."
tags: [protocolos, webdav, oidc]
---

```hcl
tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true
}
```

HTTP Basic, o un token bearer, sobre [TLS]({{< relref "/security/tls.md" >}}), a partir de
archivos o de ACME.

{{< callout type="error" >}}
**Sin cifrar no, desde la v0.9.0**

HTTP Basic es la contraseña en cada petición, y un token bearer vale tanto
como ella. WebDAV **sin TLS**, en una dirección que otras máquinas pueden
alcanzar, mientras haya alguien que se autentique, se **rechaza al
arrancar**: `serve "webdav" { addr = "0.0.0.0:8080" }`, el ejemplo que
mostraba antes esta página, ya no arranca. Cuando un proxy termina TLS
delante, indíquelo:

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true
}
```

Loopback, y un servidor sin nadie a quien autenticar, no se ven afectados.
Consulte [TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

{{< callout type="info" >}}
**404, no 403**

Un recurso compartido que una persona no puede usar responde **404**. No se
confirma que exista.
{{< /callout >}}

## Un token, por el único protocolo que puede transportarlo {#a-token-over-the-one-protocol-that-can-carry-one}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
}
```

Un navegador tiene un token y ninguna contraseña. Por eso WebDAV acepta
`Authorization: Bearer` y, donde también hay personas locales, el desafío que envía
ofrece **ambos**: el cliente elige el que puede responder. El token lo verifica
[go-authn/oidc](https://github.com/go-authn/oidc): firma, emisor, audiencia,
caducidad.

{{< callout type="error" >}}
**Un token bearer: solo WebDAV**

SMB autentica con NTLMv2, SFTP con una clave o un certificado, NFS con nada
en absoluto, y S3 con una firma SigV4: ninguno tiene dónde poner una
cabecera `Authorization`. Es un hecho de los protocolos, no un límite de este
programa.

La palabra del proveedor llega en cambio a **SFTP** dentro de un certificado:
consulte [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}). Una
configuración que nombra un proveedor y no sirve ni WebDAV ni un SFTP de ese
tipo se **rechaza en lugar de arrancar**.
{{< /callout >}}

{{< callout type="error" >}}
**Un token dice quién cree el proveedor que es alguien. No dice que este servidor tenga un recurso compartido para esa persona.**

Un token válido para un nombre que ningún origen de aquí conoce se rechaza:
la lectura segura de *no le conozco* no es *tiene permiso*. Tampoco un nombre
que también tiene una cuenta local obtiene esa cuenta, salvo que
`local_names = true` indique que los nombres del proveedor son los de este
servidor (desactivado por defecto, desde la v0.20.0); sin ello, ese token
también se rechaza, salvo que una regla `oidc:` nombre a la persona.
{{< /callout >}}

Un sitio en el que el proveedor **es** el directorio lo indica:

```hcl
oidc {
  issuer    = "https://login.example.org"
  audience  = "fileshare"
  trust_all = true      # everybody that provider vouches for, not just these people
}
```

Aquí **no hay flujo de inicio de sesión**: ni redirección, ni secreto de cliente, ni
cookies. Esto es el servidor de recursos.

**El nombre** que da un token es `preferred_username`, o el claim que nombre
`username_claim`. Sin `preferred_username`, es el correo electrónico solo si el
proveedor indica `email_verified: true`, y si no, `sub`. Con
`username_claim = "email"`, un token cuyo correo no está verificado se rechaza, tanto
por WebDAV como para un inicio de sesión opkssh por SFTP. Mientras no está
verificado, el correo es solo lo que la persona escribió, y un token firmado que lo
llevara quedaría vinculado a una dirección que nadie comprobó (OpenID Connect Core
5.1; go-authn/oidc v0.2.0, fileshare v0.16.3).

Qué personas del proveedor obtienen un recurso compartido —grupos, personas con
nombre, instituciones enteras— se escribe con reglas `oidc:` y `domains`; consulte
[las personas que nombra el proveedor de identidad]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).

## Un token sobrevive a la persona, salvo que algo diga lo contrario {#a-token-outlives-the-person-unless-something-says-otherwise}

Un token se verifica aquí, por sí solo, y es válido hasta que caduca, pase lo que
pase después con la persona. Ninguna lista de revocación puede nombrarlo. Un
[bloque `ssf`]({{< relref "/security/shared-signals.md" >}}) hace que un `session-revoked` de CAEP
procedente del proveedor invalide todos los tokens emitidos (`iat`) antes de él.
