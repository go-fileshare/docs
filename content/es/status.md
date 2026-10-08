---
title: "Estado"
weight: 60
description: "Qué versión de fileshare describen estas páginas, qué añadió cada versión, qué se ha verificado con clientes reales y qué no está implementado todavía."
tags: [estado, versiones]
---

Estas páginas describen **fileshare v0.24.0**.

## Lo que añadió cada versión {#what-each-release-added}

| | |
|---|---|
| v0.7.0 | un recurso compartido restringido por NFS, con un [bloque `kerberos`]({{< relref "/protocols/nfs.md#the-refusal-that-used-to-be-permanent" >}}) |
| v0.8.0 | la [API de administración]({{< relref "/administration/_index.md" >}}) sobre gRPC; [recursos compartidos de directorio]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}); [`/healthz`, `/readyz`, `/metrics`]({{< relref "/administration/health.md" >}}) |
| v0.9.0 | [TLS]({{< relref "/security/tls.md" >}}) para WebDAV, S3 y NFS, a partir de archivos o de ACME; **WebDAV con contraseñas rechazado sin cifrar** |
| v0.10.0 | [volver a leer el directorio]({{< relref "/administration/reload.md" >}}): `reload = "5m"`, `SIGHUP`, `ReloadDirectory` |
| v0.11.0 | [SFTP para las personas que el proveedor de identidad avala]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}): su CA SSH, y opkssh |
| v0.12.0 | revocación: una [KRL]({{< relref "/security/revocation-lists.md" >}}) para los certificados SSH del proveedor, una CRL para las [identidades NFS a partir de certificados]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.13.0 | [señales compartidas]({{< relref "/security/shared-signals.md" >}}): un `session-revoked` de CAEP invalida todo lo que el proveedor emitió antes, en todos los protocolos |
| v0.14.0 | las correcciones de la revisión de seguridad: [nombres de recurso compartido que un protocolo puede transportar]({{< relref "/administration/_index.md#what-it-will-and-will-not-touch" >}}), ningún recurso compartido que contenga la configuración o sus secretos, `source_roots` comprobado de nuevo al arrancar, `max_age` de SSF ≥ 1m y `retain` ≥ 169h |
| v0.15.0 | una [KRL procedente de `ssh_krl_url` debe estar firmada y fechada]({{< relref "/security/revocation-lists.md#signed-and-dated-from-ssh_krl_url" >}}) |
| v0.16.0 | las listas de revocación [nunca retroceden]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}}), tampoco tras un reinicio, con `ssh_krl_state_file` / `crl_state_file`; los anclajes de confianza y los archivos de lista no pueden estar dentro de un recurso compartido |
| v0.16.1 | requiere Go 1.26.6: nueve avisos alcanzables de la biblioteca estándar en 1.26.4 |
| v0.16.2 | la [firma de una CRL de NFS se comprueba antes de analizarla]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}) |
| v0.16.3 | [nadie queda nombrado por un correo electrónico no verificado]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}) |
| v0.16.4 | solo pruebas |
| v0.16.5 | solo pruebas ([#36](https://github.com/go-fileshare/fileshare/issues/36): una prueba interpretaba un error de transporte como un éxito) |
| v0.16.6 | el rechazo de NFS nombra las dos salidas: un bloque `kerberos`, o [`identity = "certificate"`]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.16.7 | go-authn/oidc v0.2.2: el tamaño de una clave RSA se cuenta en bits. Antes, una clave de 1024 bits rellenada con octetos a cero verificaba un token de WebDAV o un PK Token de opkssh |
| v0.17.0 | las correcciones de una auditoría de seguridad: un recurso compartido no puede contener otro; los archivos de acceso (`authorized_keys_file`, `dsn_file`, …) no pueden estar en un recurso compartido; WebDAV nunca pasa a anónimo porque la lectura de un directorio haya vuelto vacía; S3 respeta `protocols`; una conexión que no se ha autenticado en 30 s se cierra; una sesión opkssh termina en `opkssh_max_age`; una sesión revocada sigue revocada y se cierra |
| v0.17.1 | go-authn/oidc v0.2.4 (ningún conjunto de claves tras una redirección a http), servercert v0.3.0 (la ruta de la caché ACME comprobada como hace el StrictModes de sshd), krl v0.5.0, revocation v0.3.0 |
| v0.17.2 | go-filesystems/s3 v0.3.0: S3 ya no indica a un llamante no autenticado qué claves de acceso existen; las URL prefirmadas funcionan, duran una semana como máximo y se rechazan antes de su propia fecha |
| v0.18.0 | la CI fija Go 1.27.1 en lugar de `stable` |
| v0.18.1 | `fileshare check` advierte cuando un bloque `oidc` no tiene `domains`: [el nombre a secas de un proveedor llega a la cuenta local de ese nombre]({{< relref "/configuration/identity.md" >}}) |
| v0.19.0 | go-authn/directory v0.11.0: un bloque `users "ldap"` rechaza un [bind sin cifrar hacia otra máquina]({{< relref "/configuration/identity.md" >}}); las contraseñas se comparan como resúmenes, en tiempo constante |
| v0.20.0 | [`local_names`]({{< relref "/configuration/identity.md" >}}): los nombres del proveedor solo son nombres locales cuando el bloque `oidc` lo indica (auditoría de seguridad F4) |
| v0.21.0 | [volúmenes]({{< relref "/administration/volumes.md" >}}): un `fileshare provisioner` con privilegios crea datasets ZFS, subvolúmenes btrfs y directorios con cuota de proyecto XFS/ext4, y la API de administración sirve recursos compartidos a partir de ellos; un recurso compartido de directorio lleno da una misma respuesta en todos los protocolos; los `allowed_uids` del socket de administración |
| v0.21.1 | un [recurso compartido lleno]({{< relref "/administration/volumes.md#a-full-share" >}}) lo indica por SMB (`STATUS_DISK_FULL`) y por NFS (`NFS3ERR_NOSPC` / `NFS3ERR_DQUOT`) |
| v0.22.0 | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}): un certificado SSH solo se acepta donde su concesión de dominio (`ssh-domain-grant@core.aai.geant.org`, el perfil de CA SSH de EuroHPC) nombra este host, en modo cerrado ante el fallo; `ssh_accept_ungranted` deja entrar certificados sin concesión |
| v0.22.1 | go-filesystems/sftp v0.5.1: un certificado [vinculado a una dirección]({{< relref "/protocols/sftp.md#a-certificate-pinned-to-an-address" >}}) (`source-address`) se acepta desde esa dirección con `ssh_domains`, y en los certificados del proveedor y de OpenPubkey; antes, se rechazaba desde cualquier dirección |
| v0.22.2 | solo pruebas y documentación: `source-address` se prueba desde `::1` y desde una segunda dirección IPv4 (`127.0.0.2`), con `ssh_domains`, y en un certificado OpenPubkey; en Linux, con `FILESHARE_REQUIRE_JUDGE`, un juez ausente hace fallar la prueba en lugar de omitirla |
| v0.23.0 | un [volumen btrfs]({{< relref "/administration/volumes.md#the-size-a-client-sees" >}}) tiene el tamaño de su cuota por NFS, SMB y WebDAV (go-filesystems/nfs v0.7.0, smb v0.5.0, webdav v0.3.0 informan del tamaño en cada consulta); antes, btrfs mostraba el sistema de archivos entero. Un servidor cuyos recursos compartidos vienen de la API de administración ya no se detiene al arrancar cuando sirve NFS y aún no tiene ningún recurso |
| v0.24.0 | lecturas más rápidas: un GET WebDAV de un recurso compartido de directorio o de volumen sale con `sendfile(2)` en HTTP sin cifrar (no bajo TLS), NFS lee y escribe en bloques de 1 MiB (rtmax/wtmax), y los clientes SMB leen más de 64 KiB por petición (`CAP_LARGE_MTU`). En el runner de CI, un archivo de 256 MiB: WebDAV 6,6 GB/s, SMB 0,84, NFS 1,47 (un cliente) |

{{< callout type="warning" >}}
**Actualizar a la v0.21.0**

Un **recurso compartido de directorio lleno** responde de otra forma. `EDQUOT`
(una cuota) y `ENOSPC` son ahora un mismo error, "no space left on device
(the share is full)", en todos los recursos compartidos de directorio, no
solo en los volúmenes: un `EDQUOT` que era `500` en WebDAV y `NFS3ERR_IO` en
NFS es ahora `507` y `NFS3ERR_NOSPC`, y el texto de SFTP es esa frase, sin la
ruta. SMB sigue respondiendo acceso denegado. Consulte
[un recurso compartido lleno]({{< relref "/administration/volumes.md#a-full-share" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Actualizar a la v0.20.0**

El `alice` del proveedor ya no recibe los recursos compartidos del `alice`
local. Un sitio cuyos nombres en el proveedor son sus nombres locales añade
`local_names = true` al bloque `oidc` (y establece `domains`); el servidor
registra esa línea para cada nombre que rechaza por falta de ella.
{{< /callout >}}

{{< callout type="warning" >}}
**Actualizar a la v0.19.0**

Un bloque `users "ldap"` que apunta a `ldap://` en otra máquina sin
`start_tls = true` **ya no arranca**: use `ldaps://`, o StartTLS.
{{< /callout >}}

{{< callout type="warning" >}}
**Actualizar a la v0.17.0**

Una configuración con un recurso compartido dentro de otro, dos recursos
compartidos sobre un mismo origen, o un archivo de acceso
(`authorized_keys_file`, `dsn_file`, `bind_password_file`, el `ca_file` de
`ssf`) dentro de un recurso compartido **ya no arranca**; el error nombra el
recurso compartido y la ruta.
{{< /callout >}}

{{< callout type="warning" >}}
**Actualizar más allá de la v0.9.0**

Una configuración que sirve WebDAV sin TLS en una dirección que otras
máquinas pueden alcanzar, con alguien a quien autenticar, **ya no arranca**.
Añada `tls = true` y un bloque `tls`, o `plaintext = true` detrás de un proxy
que termine TLS. Consulte
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

## Verificado {#verified}

Son cosas que se hizo hacer a un cliente que este proyecto no escribió, no
afirmaciones sobre el código.

- **macOS** monta un recurso compartido por SMB mientras `curl` escribe en la misma
  imagen por WebDAV, y vuelve a leer lo que WebDAV escribió. La escritura de bob por
  WebDAV en un recurso compartido que solo puede leer da `403`; una contraseña
  errónea da `401`.
- **go-smb2** —un cliente que este proyecto no escribió— realiza veinte lecturas
  concurrentes por SMB mientras otras veinte pasan por WebDAV, con `-race`, en la CI.
- Las reglas de acceso se comprueban **a través de cada protocolo** y no solo en la
  configuración: alice escribe y bob no, por SMB y por WebDAV, y a NFS ni siquiera
  se le ofrece el recurso compartido.
- Los certificados SFTP se verifican frente al **propio cliente de OpenSSH**, que
  también rechaza la misma clave en cuanto su certificado se aparta.
- Las identidades NFS a partir de certificados se midieron con un **cliente Linux
  real** (núcleo 6.17, ktls-utils 0.9) en la CI de go-filesystems/nfs; así es como se
  encontraron [`MNT` sin cifrar y las trampas de `tlshd`]({{< relref "/security/nfs-certificates.md#measured-with-a-real-linux-client" >}}).
- La forma sin firmar de la KRL es una medición: OpenSSH 9.6 y 10.3 leen una KRL
  firmada y se saltan la firma (go-authn/krl, contra ambos).
- **El carril de interoperabilidad de go-authn/bridge** ([#12](https://github.com/go-authn/bridge/pull/12),
  [#17](https://github.com/go-authn/bridge/pull/17)) evalúa fileshare v0.13.0 frente
  al bridge real: la KRL, la CRL sobre NFS, tokens de WebDAV revocados mediante
  señales compartidas, un IdP desactivado por ámbito, y opkssh con señales compartidas.

## Todavía no {#not-yet}

**Escrituras por S3.** `PUT` y `DELETE` responden 403. La biblioteca sabe escribir,
pero un recurso compartido que una persona solo puede leer tiene que rechazar en el
mismo punto en que lo hace SFTP, y eso no está conectado. Rechazar es mejor que un
objeto escrito a medias.

**Un token OIDC por S3.** Los tokens funcionan por [WebDAV]({{< relref "/protocols/webdav.md" >}}) y
en ningún otro sitio. La vía habitual para S3 es STS `AssumeRoleWithWebIdentity`,
que canjea el token por credenciales temporales con las que luego firma el cliente:
un mecanismo distinto de aceptar un token bearer, y no implementado.

**`--isolate` con un bloque `admin` o `metrics`.** Rechazado: no hay un único
proceso al que se pueda aplicar un cambio de la API. Consulte
[un proceso por protocolo]({{< relref "/operations/isolation.md#not-with-an-admin-or-a-metrics-block-yet" >}}).

**Una persona por usuario en un cliente NFS Linux compartido, a partir de un
certificado.** Un cliente Linux asocia el certificado a un montaje, así que todos los
usuarios del montaje son la persona que este nombra. Es una propiedad del cliente, y
la respuesta ahí es `sec=krb5`. Consulte [lo que no puede prometer]({{< relref "/security/nfs-certificates.md#what-it-cannot-promise" >}}).

[S3 en sí]({{< relref "/protocols/s3.md" >}}) ya está publicado: un recurso compartido es un bucket, servido
sobre el mismo árbol por usuario que usa SFTP.

## Medido, y presentado como medido {#measured-and-stated-as-measured}

Afirmaciones de estas páginas que son un seguro o una medición, y no la reproducción
de un defecto, y que también están escritas así en el código fuente:

- Ocho goroutines que escriben a través de un controlador fat32 sin envolver, con
  `-race`, no producen hoy ninguna condición de carrera. El [bloqueo compartido]({{< relref "/operations/locking.md" >}}) es un seguro
  sobre un contrato —`go-filesystems/interface` no promete nada sobre llamadas
  concurrentes basadas en rutas—, no la corrección de un fallo observado.
- Los [tamaños según las etiquetas de compilación]({{< relref "/operations/build-tags.md" >}}) se midieron, incluidos los
  13,2 MB que cuesta `hashicorp/go-plugin` por sí solo, lo que decidió en contra de
  un framework de plugins, y los 11,7 MB que cuesta gRPC ahora que la API de
  administración lo trae, que es la razón de ser de `nogrpc`.
- Se diseñó una extensión de certificado propia para etiquetas de grupo y se
  **midió** frente al analizador X.509 de Go, que rechazó el certificado entero; los
  grupos viajan en su lugar como [URI de etiqueta]({{< relref "/security/nfs-certificates.md#what-the-certificate-carries" >}}).
