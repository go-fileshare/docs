---
title: "Revocar lo que ninguna lista cubre: las señales compartidas"
linkTitle: "Señales compartidas"
weight: 40
description: "Revocar tokens de acceso, certificados OpenPubkey y otras credenciales federadas con eventos CAEP session-revoked procedentes de un transmisor de señales compartidas."
tags: [seguridad, revocación, señales compartidas]
---

Un certificado tiene una lista de revocación: la [KRL]({{< relref "/security/revocation-lists.md" >}}),
la [CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}). Un **token de
acceso** que fileshare verifica por sí solo, y un **certificado OpenPubkey** firmado
por la propia clave de la persona, no la tienen: son válidos hasta que caducan, pase
lo que pase después con la persona.

Desde la v0.13.0, el [OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html)
cubre ese hueco, con un evento **`session-revoked`** de
[CAEP](https://openid.net/specs/openid-caep-1_0-final.html):

```hcl
ssf {
  transmitter        = "https://bridge.example.org"          # its issuer
  audience           = "https://files.example.org"           # what this server is to it
  client_id          = "fileshare"                           # OAuth client credentials,
  client_secret_file = "/etc/fileshare/ssf.secret"           # scope "ssf" (or token_file)
  state_file         = "/var/lib/fileshare/revocations.json"
  max_age            = "10m"                                 # default; retain = "192h"
}
```

go-authn/bridge envía uno cuando se desactiva una persona o un IdP; fileshare lo
**sondea** (RFC 8936, el modo por defecto de SSF) y guarda, para cada persona,
**cuándo** ocurrió.

## Todo lo emitido antes se rechaza, en todos los protocolos {#everything-issued-before-is-refused-on-every-protocol}

A partir de ese momento, toda credencial federada **emitida antes** se rechaza, y
**las sesiones que esas credenciales abrieron dejan de recibir servicio**, a través
de los mismos puntos de control que los de la KRL. Lo que significa «emitida»
depende de lo que la transportaba:

| protocolo | credencial | su momento de emisión |
|---|---|---|
| WebDAV | un token de acceso | su `iat` |
| SFTP | un certificado firmado por la CA SSH del proveedor | el inicio de su validez (`ValidAfter`) |
| SFTP | un certificado OpenPubkey (opkssh) | el `iat` del ID token |
| NFS | un certificado de cliente ([identity = "certificate"]({{< relref "/security/nfs-certificates.md" >}})) | su `NotBefore` |

Lo que el proveedor emite **después** es suyo: una persona reactivada no queda
bloqueada.

{{< callout type="info" >}}
**El mismo segundo cuenta como «antes»**

El `event_timestamp` de CAEP y el `iat` de un token son segundos enteros, y
una credencial emitida **en el momento de la revocación o antes** se
rechaza; así, una emitida en el mismo segundo que una revocación también se
rechaza, y una persona reactivada justo después de ser desactivada obtiene
credenciales válidas a partir del segundo siguiente. Es el lado
conservador, a propósito. Un momento de emisión desconocido no equivale a
«hace mucho»: se rechaza en cuanto existe alguna revocación para esa
persona.
{{< /callout >}}

## A quién se refiere un evento {#who-an-event-is-about}

El sujeto es un **`account`** del RFC 9493 (`acct:user@domain`, el nombre que usan
los recursos compartidos), un **`iss_sub`**, un **`email`**, o **`aliases`** de
ellos.

**Un IdP desactivado por completo** llega como un sujeto *tenant* de CAEP, con los
ámbitos (scopes) del IdP en el evento: todas las personas cuyo nombre está `@` en uno
de ellos —comparado entero: `evil-univ-a.fr` no es `univ-a.fr`— y cuya credencial
se emitió antes se rechazan, **incluidas las personas que el proveedor ya no
recuerda**.

## Cómo se autentica el receptor {#how-the-receiver-authenticates}

Con **credenciales de cliente OAuth** (RFC 6749 §4.4, ámbito `ssf`): `client_id` y
`client_secret_file`, con el token obtenido del propio endpoint de tokens del
transmisor —que se encuentra en su configuración OpenID, o en `token_url`— y
renovado antes de que caduque. `token_file` existe en su lugar para los
transmisores que entregan un token bearer de larga duración. Uno u otro: ambos, o
ninguno, se rechaza.

| campo | por defecto | |
|---|---|---|
| `transmitter` | | su emisor, **solo HTTPS** |
| `audience` | | lo que este servidor es para el transmisor; obligatorio, o aquí se creería un SET dirigido a otro receptor |
| `client_id`, `client_secret_file` | | credenciales de cliente OAuth, juntas |
| `token_url` | descubierto | solo HTTPS, o el secreto cruza la red sin cifrar |
| `token_file` | | un token de larga duración en lugar de credenciales de cliente |
| `state_file` | | **obligatorio**: dónde se anotan las revocaciones |
| `ca_file` | las del sistema | fija las autoridades del transmisor |
| `max_age` | `10m` | cuánto tiempo sin noticias del transmisor antes de que se rechacen las credenciales federadas; al menos un minuto |
| `retain` | `192h` | cuánto tiempo se conserva una revocación; al menos 169h (desde la v0.14.0) |

Una revocación **se anota antes de confirmarse**, así que una revocación confirmada
sobrevive a un reinicio; y se conserva durante `retain`, más tiempo que cualquier
credencial que pudiera invalidar (192h supera la credencial de mayor duración que
emite el proveedor, un certificado SSH de 168h).

Un bloque `ssf` sin nada federado que revocar —sin bloque `oidc`, sin
`identity = "certificate"` en NFS— se rechaza.

El transporte es [github.com/hstern/go-ssf](https://github.com/hstern/go-ssf): el
descubrimiento, el sondeo del RFC 8936, la capa JWS del SET. Lo que un evento
**significa** lo decide fileshare.

## Falla en modo cerrado {#it-fails-closed}

{{< callout type="error" >}}
**Mientras el transmisor calla, las credenciales federadas se rechazan**

Como con la KRL: mientras el transmisor no haya respondido a un sondeo
dentro de `max_age`, se rechazan **todas las credenciales federadas**,
porque «no ha llegado ninguna revocación» y «no podía llegar ninguna» tienen
el mismo aspecto. `fileshare_ssf_last_heard_seconds` es la métrica sobre la
que alertar; `fileshare_ssf_revoked_subjects` indica cuántas revocaciones se
conservan. Consulte
[salud y métricas]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

## Comprobado de extremo a extremo {#checked-end-to-end}

El carril de interoperabilidad de go-authn/bridge
([#12](https://github.com/go-authn/bridge/pull/12),
[#17](https://github.com/go-authn/bridge/pull/17)) evalúa fileshare v0.13.0 frente
al bridge real: la KRL, la CRL sobre NFS, tokens de WebDAV revocados mediante SSF,
un IdP desactivado por ámbito, y opkssh mediante SSF, el caso al que no llega
ninguna lista de revocación.
