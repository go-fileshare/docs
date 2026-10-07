---
title: "TLS, y certificados de ACME"
weight: 10
description: "Servir WebDAV, S3 y NFS sobre TLS con un certificado a partir de archivos o de una CA ACME, y por qué WebDAV con contraseñas se rechaza sin cifrar."
tags: [seguridad, tls, acme]
---

Desde la v0.9.0, **WebDAV y S3 se sirven por HTTPS, y NFS sobre RPC-with-TLS**
(RFC 9289), con `tls = true` en su bloque `serve` y un bloque `tls` que indica de
dónde procede el certificado.

## A partir de archivos {#from-files}

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

Los archivos **se recargan cuando cambian**, así que una renovación escrita por otra
herramienta se aplica sin reiniciar. El certificado lo gestiona
[go-authn/servercert](https://github.com/go-authn/servercert).

## A partir de una CA ACME {#from-an-acme-ca}

Let's Encrypt por defecto; o una CA que le conoce mediante **vinculación de cuenta
externa** (external account binding, EAB), como GÉANT TCS a través de HARICA:

```hcl
tls {
  acme {
    directory_url     = "https://acme-v02.harica.gr/acme/<uuid>/directory"   # the Server URL cm.harica.gr shows
    domains           = ["files.example.org"]
    cache_dir         = "/var/lib/fileshare/acme"
    eab_key_id        = "…"
    eab_hmac_key_file = "/etc/fileshare/tls/eab.key"   # a secret: a file, never the config
  }
}
```

| campo | |
|---|---|
| `directory_url` | el directorio ACME de la CA; vacío es Let's Encrypt |
| `domains` | los nombres que se solicitan (obligatorio) |
| `cache_dir` | dónde se guardan la cuenta y los certificados (obligatorio) |
| `email` | el contacto de la cuenta, opcional |
| `eab_key_id`, `eab_hmac_key_file` | vinculación de cuenta externa; la clave HMAC es un secreto, así que se lee de un archivo, en base64url, tal como la entrega la CA (también se lee base64 estándar); cualquier otra cosa se rechaza |
| `http_challenge` | dónde responder al desafío http-01, p. ej. `"0.0.0.0:80"` |

### Que ACME pueda funcionar depende de cómo comprueba la CA el nombre {#whether-acme-can-work-is-decided-by-how-the-ca-checks-the-name}

| desafío | la CA se conecta a | por tanto |
|---|---|---|
| tls-alpn-01 (RFC 8737) | el puerto **443** | debe servirse un protocolo TLS en el 443 |
| http-01 (RFC 8555 §8.3) | el puerto **80** | `http_challenge = "0.0.0.0:80"` lo responde, o la dirección a la que se reenvía un puerto 80 |
| ninguno | nada | una cuenta de CA con el dominio **prevalidado** no pide ningún desafío: las cuentas EAB empresariales de HARICA para GÉANT TCS; así, un servidor **que nadie de fuera puede alcanzar** obtiene igualmente su certificado |

La última fila es la que importa para un servidor de archivos dentro de una red de
campus: con una cuenta GÉANT TCS cuyo dominio la institución ya ha validado en
HARICA, no hay que abrir ningún puerto a Internet.

Un certificado se solicita en la **primera conexión TLS que nombra el host** (SNI);
un cliente que se conecta por dirección IP no obtiene ninguno.

{{< callout type="info" >}}
**ALPN**

Con ACME, la lista ALPN del servidor contiene `acme-tls/1`, y un servidor Go
rechaza a un cliente cuya lista no tiene nada en común con la suya. Por eso
se **añade** `http/1.1` para WebDAV y S3 —todos los navegadores y clientes
WebDAV lo ofrecen— y `sunrpc` para NFS, tal como el RFC 9289 §5.2 identifica
RPC-with-TLS. No `h2`: HTTP/2 no se ofrece en estas escuchas, y ofrecerlo
sería mentir. `acme-tls/1` permanece en todas las escuchas, así que
tls-alpn-01 se responde allí donde llegue la CA.
{{< /callout >}}

## WebDAV no se sirve sin cifrar {#webdav-is-not-served-in-the-clear}

{{< callout type="error" >}}
**CAMBIO INCOMPATIBLE en la v0.9.0: WebDAV con contraseñas en una dirección alcanzable se rechaza**

HTTP Basic es la contraseña, en base64, en cada petición, y un token bearer
vale tanto como ella. Así que una configuración que sirve WebDAV **sin TLS**
en una dirección que otras máquinas pueden alcanzar, mientras haya alguien
que se autentique, se **rechaza, no se advierte**: una advertencia pasa
desapercibida, y la contraseña no vuelve:

```
webdav on 0.0.0.0:8080 would carry passwords in the clear: serve it with
tls = true, or -- when TLS is terminated in front of it, by a proxy -- say
plaintext = true
```
{{< /callout >}}

Una configuración que arrancaba antes de la v0.9.0 con
`serve "webdav" { addr = "0.0.0.0:8080" }` no arranca después. Dos salidas:

```hcl
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true            # with a tls block
}
```

o, cuando un proxy inverso termina TLS delante y reenvía sin cifrar:

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true       # TLS is terminated in front of this, on purpose
}
```

Las direcciones de **loopback** (`127.0.0.1`, `::1`, `localhost`) y un servidor
**sin nadie a quien autenticar** —sin `user`, sin `users`, sin bloque `oidc`— no se
ven afectados. Un bloque `serve` sin `addr` queda en loopback, y tampoco se ve
afectado.

`plaintext` es solo para WebDAV, el protocolo que envía una contraseña, y `plaintext`
junto con `tls` se rechaza: uno u otro.

## Lo que dice `check` {#what-check-says}

[`fileshare check`]({{< relref "/configuration/check.md" >}}) imprime, por protocolo, lo que
está cifrado y lo que va sin cifrar **a propósito**:

```
webdav: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem
nfs: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem; clients present one signed by /etc/fileshare/tls/clients.pem (the machine, not the person)
```

y, para WebDAV detrás de un proxy,

```
webdav: in the clear on 10.0.0.5:8080, on purpose (plaintext = true): TLS must be terminated in front of it
```

## Por protocolo {#per-protocol}

| protocolo | `tls = true` | |
|---|---|---|
| WebDAV | sí | HTTPS |
| S3 | sí | HTTPS |
| NFS | sí | RFC 9289; véase más abajo |
| SMB | **rechazado** | SMB 3 cifra con sus propias claves |
| SFTP | **rechazado** | SFTP es SSH |

## NFS sobre TLS prueba la máquina, no la persona {#nfs-over-tls-proves-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

`client_ca_file` obliga al cliente a presentar un certificado de esa autoridad
—**qué máquina está montando**—, y el RFC 9289 deja la autenticación de usuarios
como estaba: el uid de dentro sigue siendo el que afirma `AUTH_SYS`. Así que un
recurso compartido que nombra quién puede usarlo **sigue rechazándose por NFS** sin
un bloque `kerberos`, o sin
[identidades a partir de certificados]({{< relref "/security/nfs-certificates.md" >}}), que son un uso
distinto del mismo certificado.

TLS **se ofrece, no se exige**: a un cliente que nunca lo pide se le siguen sirviendo
los recursos compartidos abiertos. `client_ca_file` sin `tls = true` se rechaza —no
comprobaría nada—, y `client_ca_file` en cualquier protocolo distinto de NFS también
se rechaza.

## Qué más se rechaza {#what-else-is-refused}

- `tls = true` sin bloque `tls`: nada indica de dónde procede el certificado.
- Un bloque `tls` que ningún bloque `serve` usa: nada lo usaría.
- Un `http_challenge` que no es una dirección en la que escuchar.
