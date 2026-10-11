---
title: "La API de administración"
linkTitle: "Administración"
weight: 30
description: "La API de administración gRPC que crea, modifica, desactiva y concede recursos compartidos mientras el servidor está en marcha, dónde escucha y qué rechaza."
tags: [administración, api de administración, grpc]
---

Desde la v0.8.0, los recursos compartidos pueden cambiar mientras el servidor está en
marcha. Un bloque `admin` activa un servicio gRPC que crea, modifica y retira de
servicio recursos compartidos sin reiniciar y sin editar ningún archivo.

```hcl
admin {
  listen       = "unix:///run/fileshare/admin.sock"   # made 0600
  state_file   = "/var/lib/fileshare/shares.json"
  source_roots = ["/srv/images", "/data"]
}
```

Nada escucha salvo que se escriba el bloque.

## Lo que hace {#what-it-does}

El servicio es
[`fileshare.admin.v1.AdminService`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto):

| | |
|---|---|
| `CreateShare`, `UpdateShare`, `DeleteShare` | definir un recurso compartido a partir de una imagen, un directorio o (desde la v0.21.0) un volumen; cambiar `read_only` o `protocols` (su origen no puede cambiar: elimínelo y cree otro) |
| `DisableShare`, `EnableShare` | retirar un recurso compartido de servicio y volver a ponerlo |
| `Grant`, `Revoke` | dar a un sujeto acceso de lectura o de escritura, o retirárselo |
| `ListShares`, `GetShare` | todos los recursos compartidos, con lo que sirven, por qué protocolos, y por qué no por los demás |
| `ListUsers`, `ListGroups` | a quién puede nombrar un permiso; cada usuario con los protocolos a los que pueden responder sus credenciales |
| `ReloadDirectory` | volver a leer las personas, ahora; consulte [volver a leer el directorio]({{< relref "/administration/reload.md" >}}) |
| `GetServerInfo` | el nombre, la versión, la hora de arranque, la generación y las escuchas |
| `ListParents`, `CreateVolume`, `ResizeVolume`, `SnapshotVolume`, `DeleteVolume`, `GetVolume`, `ListVolumes` | almacenamiento creado mediante un aprovisionador con privilegios, y recursos compartidos servidos a partir de él; consulte [volúmenes]({{< relref "/administration/volumes.md" >}}) (desde la v0.21.0) |

Un permiso nombra a un **usuario**, un **grupo** (`@group` en un archivo de
configuración), un **valor `oidc:groups:`** o un **nombre `oidc:user:`**: el
vocabulario que usa el archivo de configuración; consulte
[las personas que nombra el proveedor de identidad]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).

`grpc.health.v1` responde en la misma escucha. `reflection = true` en el bloque
activa la reflexión del servidor gRPC, para `grpcurl`.

### Cada cambio indica lo que hizo al servirlo {#every-change-says-what-serving-it-did}

Un cambio responde con un mensaje `Applied`:

```proto
message Applied {
  uint64 generation = 1;          // the generation now being served
  uint64 connections_closed = 2;  // how many the previous one had open, and closed
}
```

Por eso cada RPC tiene un mensaje de respuesta propio en lugar de devolver el propio
recurso compartido: el `.proto` sigue las reglas de lint `STANDARD` de
[buf](https://buf.build), comprobadas por `buf lint` en la CI de fileshare, por
encima del «devolver el recurso» de AIP-131; el envoltorio es lo que transporta
`Applied`.

## Dónde escucha, y quién puede llamarla {#where-it-listens-and-who-may-call-it}

{{< callout type="error" >}}
**Por TCP, TLS mutuo o nada**

`tls_cert_file`, `tls_key_file` y `client_ca_file`, los tres, **loopback
incluido**: cualquier usuario local puede alcanzar loopback, y esta API
decide quién lee los archivos de quién. Un socket unix se crea con **0600**,
y sus permisos son su control de acceso.
{{< /callout >}}

```hcl
admin {
  listen         = "127.0.0.1:7443"
  tls_cert_file  = "/etc/fileshare/admin/server.pem"
  tls_key_file   = "/etc/fileshare/admin/server.key"
  client_ca_file = "/etc/fileshare/admin/clients.pem"
  state_file     = "/var/lib/fileshare/shares.json"
  source_roots   = ["/srv/images"]
}
```

La escucha es
[grpc-transports/control](https://github.com/grpc-transports/control). Cada cambio
se registra con quién lo hizo: el CN del certificado de cliente, o el uid del par del
socket.

## Por HTTPS, para tokens OIDC {#over-https-for-oidc-tokens}

Desde la v0.28.0 la misma API — el mismo estado, la misma auditoría — se sirve
también por HTTPS cuando el bloque `admin` tiene un bloque `web`: **Connect,
gRPC-Web y gRPC en un solo puerto**, con el certificado del
[bloque `tls`]({{< relref "/security/tls.md" >}}). Es con ella con la que hablan
las interfaces web y nativas.

```hcl
admin {
  listen     = "unix:///run/fileshare/admin.sock"
  state_file = "/var/lib/fileshare/shares.json"
  web {
    listen = "0.0.0.0:8443"
    issuer "https://login.example.org" {
      audience = "fileshare-a"            # what THIS server is called there
      groups   = ["fileshare-admins"]
    }
    issuer "https://login.partner.example.org" {
      audience = "fileshare-a.partner"
      subjects = ["5b0c…"]                # a person is (issuer, sub)
    }
  }
}
```

| campo | |
|---|---|
| `listen` | una dirección TCP; rechazado sin bloque `tls` — un token portador vale lo que una contraseña |
| `issuer "<url>"` | un bloque por proveedor de identidad, su `iss`; dado dos veces se rechaza |
| `audience` | el `aud` que un token debe nombrar: **cómo se llama este servidor** en ese proveedor; obligatorio |
| `subjects`, `groups` | quién puede llamar: un `sub`, o un grupo del claim de grupos; al menos uno de los dos |
| `jwks_url` | omite el descubrimiento OIDC, para un proveedor que no lo publica |
| `groups_claim` | el claim que lleva los grupos; `groups` por defecto |

**A quién se responde.** Una llamada se atiende cuando el verificador de un
emisor acepta su token portador — firma, `iss`, un `aud` que nombra este
servidor, fechas — **y** el bloque de ese emisor nombra el sujeto del token o
uno de sus grupos. Un sujeto inscrito en un emisor no es nadie en otro: una
persona es (emisor, `sub`). La comprobación se hace solo con las cabeceras,
antes de decodificar un byte del mensaje.

**Una audiencia por servidor.** Un token dirigido a varios servidores podría
ser reenviado por cualquiera de ellos a los demás (RFC 8707, RFC 9068): cada
servidor tiene la suya, y una interfaz que maneja varios servidores pide un
token por servidor.

**Lo que sabe quien es rechazado.** «Rechazado», nada más. El motivo, y cada
cambio hecho, van a la auditoría como `oidc=<emisor> <sub>`.

**Sin CORS.** Los navegadores no la llaman directamente: llegan a ella a través
del servidor de las interfaces, que guarda los tokens. Las peticiones se
limitan a 1 MiB y cada fase de una conexión tiene un plazo. Cada emisor se
contacta **al arrancar**, para leer sus claves; uno inalcanzable detiene el
arranque, nombrándolo.

### En un navegador: la consola {#in-a-browser-the-console}

[go-fileshare/console](https://github.com/go-fileshare/console) es la consola
de administración que llama a esta API: recursos compartidos, detalles,
permisos, crear, activar, desactivar, borrar, conceder y retirar, y
[volúmenes]({{< relref "/administration/volumes.md" >}}) (crear, redimensionar,
instantánea, compartir, borrar -- sin destruir nunca datos), en todos los
servidores a la vez. Funciona en un navegador detrás de
[go-fileshare/portal](https://github.com/go-fileshare/portal), que autentica a
las personas con OpenID Connect y guarda sus tokens: uno por servidor, dirigido
solo a ese servidor. El navegador nunca tiene ninguno.

```hcl
# portal.hcl
server "paris" {
  url      = "https://fs-paris.example.org:8443"   # este puerto web
  issuer   = "https://login.example.org"
  resource = "https://fs-paris.example.org/"       # la audiencia de este servidor, arriba
}
ui "console" { dir = "/usr/share/fileshare-console" }
```

La misma consola funciona también **de forma nativa**, sin portal. Se autentica
con el navegador del sistema y un puerto local, como cliente público
([go-authn/rp](https://github.com/go-authn/rp), RFC 8252), y llama a cada
servidor con un token solo para él. Los servidores se listan en su propio
`console.hcl`, con los mismos `url` y `resource` que aquí.

El **explorador de archivos**, [go-fileshare/explorer](https://github.com/go-fileshare/explorer),
funciona detrás del mismo portal: navegar, subir, descargar, renombrar, copiar y
borrar, en cada recurso compartido que una persona puede usar. Llega al WebDAV
de cada servidor a través del portal (`/dav/<servidor>/...`) con un token para
ese servidor, y el portal sirve lo que vuelve **inerte**
(`Content-Security-Policy: sandbox`, como adjunto): una página guardada en un
recurso nunca se ejecuta en nombre del portal. Necesita fileshare **v0.29.0** o
posterior, cuyo [WebDAV]({{< relref "/protocols/webdav.md#the-root-and-uploads-in-pieces" >}})
lista los recursos en su raíz y acepta una subida por partes.

```hcl
# portal.hcl, el mismo servidor
server "paris" {
  # ... como arriba, y su WebDAV:
  dav_url = "https://fs-paris.example.org"
}
ui "explorer" { dir = "/usr/share/fileshare-explorer" }
```

El explorador funciona también **de forma nativa** (explorer v0.2.0), sin
portal, como la consola: se autentica con el navegador del sistema como cliente
público y llama al WebDAV de cada servidor con un token solo para él. Sus
servidores se listan en `explorer.hcl`, cada uno con su `dav_url` (solo https)
y la `resource` de su bloque `oidc`. Ninguno de los dos clientes nativos sigue
una redirección con un token: el cliente HTTP de Go conserva `Authorization` en
una redirección al mismo host con cualquier esquema, lo que un proxy mal
configurado que respondiera `307 http://...` convertiría en un token enviado en
claro.

La `audience` de cada servidor es la `resource` que el portal usa para él. No
hace falta escribir una consola propia para usar la API: sirve cualquier
cliente Connect, gRPC-Web o gRPC con un token para este servidor.

## Lo que tocará y lo que no {#what-it-will-and-will-not-touch}

**Gestiona los recursos compartidos que ha creado.** Un recurso compartido escrito en
la configuración aparece en la lista, con los mismos campos, y un cambio en su
definición se rechaza con `FAILED_PRECONDITION`: un recurso compartido definido en
dos sitios es una pregunta que nadie quiere responder, y el archivo es donde se
define ese. Los recursos compartidos de la API viven en `state_file`, escrito de
forma atómica, y se vuelven a servir en el siguiente arranque; por eso se rechaza un
bloque sin `state_file`.

**Un origen debe estar bajo `source_roots`**, una vez resuelto —enlaces seguidos,
`..` eliminados—, y lo que se guarda es la ruta resuelta. Sin `source_roots`, la API
**no puede crear ningún recurso compartido**: el proceso puede leer `/dev` y `/etc`,
y nadie pretendía entregárselos a un llamante. Se vuelve a comprobar en cada
arranque, y un recurso compartido de `state_file` que ya no está bajo una raíz
detiene el arranque, nombrándolo.

**Todo recurso compartido tiene al menos un permiso.** Un recurso compartido sin
ninguno está abierto a cualquiera que se autentique. El archivo puede decirlo a
propósito; una llamada a la API no debería decirlo por omisión. Por eso
`CreateShare` necesita un permiso, y revocar el último se rechaza: elimine el recurso
compartido en su lugar.

**Nombres que un protocolo puede transportar.** Un nombre de recurso compartido
tiene como máximo 80 caracteres, no empieza ni termina con un espacio, no es `.` ni
`..`, y no contiene ningún carácter no imprimible ni ninguno de `:*?"<>|{}%`. Los
sujetos tampoco contienen caracteres no imprimibles, y un recurso compartido admite
como máximo 1000 permisos: mucho antes, eso es trabajo para un grupo. Cada uno de
estos casos se aceptaba antes de la v0.14.0, se escribía en `state_file`, y luego era
fatal en cada arranque.

**Ningún recurso compartido puede contener** la configuración, el archivo de estado,
ni los secretos que nombran: quien escribiera en él reescribiría quién puede hacer
qué. Desde la v0.17.0, eso incluye `authorized_keys_file`, el `dsn_file` y el
`bind_password_file` de un bloque `users`, el `ca_file` del bloque `ssf`, y una base
de datos sqlite que nombre un DSN: quien pudiera añadir su clave al
`authorized_keys` de alguien iniciaría sesión como esa persona. **Tampoco un recurso
compartido puede contener otro recurso compartido** (consulte
[directorios]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})).

**Un cambio se comprueba como una configuración, se abre, se anota y luego se
sirve.** Un cambio que el servidor no puede respetar —una imagen que no se abre, un
nombre que nadie tiene en el directorio— se rechaza, y lo que se servía y lo que
estaba escrito quedan como estaban.

## Desactivar un recurso compartido {#disabling-a-share}

**Desactivar es el `available = no` de Samba**: el recurso compartido sigue definido
y todo intento de conexión falla; sus conexiones abiertas se cierran y su imagen o
directorio **se libera**, de modo que el archivo puede sustituirse mientras está
fuera de servicio. `EnableShare` lo vuelve a servir, y se rechaza cuando su origen ya
no se puede abrir. `CreateShare` también puede crear un recurso compartido ya
desactivado.

A diferencia de cualquier otro cambio, se aplica también a un recurso compartido
**de la configuración** —retirar de servicio un recurso compartido es una operación,
no una definición— y sobrevive a un reinicio: se guarda en el archivo de estado.
[`fileshare check`]({{< relref "/configuration/check.md" >}}) enumera lo que está fuera de
servicio.

## Un cambio es una nueva generación {#a-change-is-a-new-generation}

{{< callout type="error" >}}
**Un cambio reinicia los servidores de protocolo, y cierra sus conexiones**

SMB comprueba quién puede conectarse una vez por conexión de árbol, y SFTP
construye el árbol de una persona una vez por inicio de sesión, así que una
sesión que sobreviviera a una revocación conservaría el acceso recién
retirado. «Revocado, salvo para quien ya estuviera conectado» no es una
revocación.
{{< /callout >}}

Así que un cambio no se aplica *a* los servidores en ejecución: estos se sustituyen
por otros nuevos construidos a partir de la nueva lista —una **generación**—, como
haría un reinicio, sin renunciar a los puertos. Las bibliotecas no permitían otra
cosa: go-filesystems/smb tiene `Share` y no `Unshare`, nfs tiene `Export` y no
`Unexport`.

Los puertos siguen vinculados, así que un cliente que se conecta durante un cambio
**espera** en lugar de ser rechazado. Las imágenes cuyo recurso compartido no cambió
conservan su controlador, sin abrirse una segunda vez. Los clientes se vuelven a
conectar; ese es el precio, pagado en cada cambio, y por eso la API aplica un cambio
por llamada y no uno por campo.

## Límites {#limits}

- `--isolate` todavía no es compatible con un bloque `admin` —ni con un bloque
  `metrics`—: no hay un único proceso al que se pueda aplicar un cambio. La
  combinación se rechaza.
- Un binario compilado con `-tags nogrpc` no tiene API de administración, y una
  configuración con un bloque `admin` se **rechaza** en lugar de servirse sin él.
  Esa etiqueta ahorra 11,7 MB; consulte
  [compilar solo lo que necesita]({{< relref "/operations/build-tags.md#the-admin-api-and-nogrpc" >}}).

## Modificar el `.proto` {#changing-the-proto}

El código Go de `proto/` se genera y se incluye en el repositorio, así que
`go install` no necesita `protoc`. Tras modificar un `.proto` —el de la API de
administración, o el `proto/fileshare/provision/v1/provision.proto` del
aprovisionador—, regenere ambos con las versiones que fija la CI de fileshare
(protoc 34.1, protoc-gen-go v1.36.12, protoc-gen-go-grpc v1.6.2). La CI los
regenera y falla cuando el código incluido difiere, o cuando se dejó código generado
sin incluir:

```sh
protoc -I proto --go_out=. --go_opt=module=github.com/go-fileshare/fileshare \
  --go-grpc_out=. --go-grpc_opt=module=github.com/go-fileshare/fileshare \
  proto/fileshare/admin/v1/admin.proto proto/fileshare/provision/v1/provision.proto
```
