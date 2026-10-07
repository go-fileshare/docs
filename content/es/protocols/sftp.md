---
title: "SFTP — claves, o certificados"
linkTitle: "SFTP: claves, o certificados"
weight: 30
description: "SFTP con claves públicas o certificados SSH: autoridades locales y del proveedor, OpenPubkey, concesiones de dominio, source-address y revocación."
tags: [protocolos, sftp, ssh]
---

Todos los clientes ya lo tienen: `sftp` viene con OpenSSH, el Finder y GNOME lo
montan, los editores lo hablan. Es también el único protocolo de los aquí descritos
cuya forma no encaja: una persona inicia sesión y llega a **un** sistema de archivos,
no a una lista de recursos compartidos.

Así que los recursos compartidos se convierten en los directorios de nivel superior
de un árbol construido para quien acaba de autenticarse:

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

Un recurso compartido que alice no puede usar **no es un directorio que alice pueda
ver**, y un recurso compartido que solo puede leer rechaza sus escrituras *en el
árbol*, antes de llegar a un controlador que las habría permitido.

{{< callout type="info" >}}
**Un renombrado entre dos recursos compartidos se rechaza**

Sería una copia y un borrado sobre dos imágenes, y eso no es lo que un
renombrado promete en ningún sitio.
{{< /callout >}}

## Configuración {#configuration}

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # optional

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # nothing here: her certificate is her credential
```

Con `trusted_user_ca_file`, basta un certificado firmado por esa autoridad que nombre
al usuario entre sus principales; así, el acceso de una persona **se emite y caduca
en otro lugar**, y no se edita ningún archivo de aquí cuando alguien llega o se va.

La firma, el periodo de validez y los principales los comprueba el `CertChecker` de
`x/crypto/ssh`; verificado frente al **propio cliente de OpenSSH**, que también
rechaza la misma clave en cuanto su certificado se aparta.

{{< callout type="warning" >}}
**Sin `host_key_file` se genera una identidad nueva en cada arranque**

El servidor lo indica, y todo cliente que ya la haya visto advertirá de una
clave cambiada, que es el cliente haciendo su trabajo.
{{< /callout >}}

## Certificados destinados a este host: la concesión de dominio {#certificates-meant-for-this-host-the-domain-grant}

Desde la v0.22.0. La [CA SSH de la EuroHPC Federation Platform](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-overview/)
(GÉANT / MyAccessID) firma certificados en los que confían todos los sitios de la
federación. Cada certificado indica a qué hosts está destinado, en la extensión
`ssh-domain-grant@core.aai.geant.org`. sshd ignora esa extensión, y fileshare también
lo hacía hasta la v0.22.0: un certificado concedido para las máquinas de otro valía
igual aquí. Con `ssh_domains`, la concesión se lee, se analiza y se compara mediante
[go-authn/sshcert](https://github.com/go-authn/sshcert), tal como dice la
especificación. Un certificado cuya concesión no nombra ninguno de los nombres de
este host se rechaza.

```hcl
trusted_user_ca_file = "/etc/fileshare/efp-ssh-ca.pub"
ssh_domains          = ["files.example.org", "sftp.example.org"]
# ssh_accept_ungranted = true   # only if a trusted CA never writes a grant
```

| clave | |
|---|---|
| `ssh_domains` | los nombres de este host, tal como los nombra una concesión. Un certificado se acepta si uno de los patrones de su concesión coincide con uno de ellos. `*` es **uno o más caracteres dentro de una misma etiqueta**: `*.example.org` concede `files.example.org` pero no `a.files.example.org`, y `files-*.example.org` no concede `files-.example.org`. La comparación no distingue mayúsculas de minúsculas. Son nombres de host, no patrones, cada uno listado una sola vez, y al menos una autoridad (`trusted_user_ca_file`, o el `ssh_ca_file` del bloque `oidc`) debe ser de confianza. |
| `ssh_accept_ungranted` | deja entrar un certificado **sin** concesión. Desactivado por defecto; sin `ssh_domains` se rechaza por carecer de sentido. |

| certificado | `ssh_domains` | `+ ssh_accept_ungranted` | sin `ssh_domains` |
|---|---|---|---|
| concedido para uno de `ssh_domains` | entra | entra | entra |
| concedido solo para otros hosts, o `[]` | rechazado | rechazado | entra |
| sin concesión | rechazado | entra, registrado | entra |
| una concesión que no se puede analizar (`null`, `[1]`, no compacta...) | rechazado | **rechazado** | entra |

Sin `ssh_domains` no se lee nada y nada cambia. La concesión se lee en los
certificados que firmó una **autoridad**: los de `trusted_user_ca_file` y los del
proveedor (`ssh_ca_file`). Un certificado OpenPubkey lo firma la propia clave del
usuario, así que una concesión en él sería la palabra del usuario sobre sí mismo. Su
audiencia es en cambio `opkssh_client_id`. Las claves simples no llevan concesión y
no se ven afectadas.

Cada rechazo se registra con el número de serie del certificado, su key ID y el
motivo:

```
sftp: alice: certificate 42 ("alice@efp") refused: its domain grant [login.example.eu] names none of this host's names [files.example.org sftp.example.org]
sftp: alice: certificate 43 ("staff") refused: it has no domain grant (ssh-domain-grant@core.aai.geant.org), and ssh_domains requires one
```

Cada decisión se cuenta también en `fileshare_sftp_domain_grant_total{result}`, con
el resultado `granted`, `accepted_ungranted`, `refused_not_granted`,
`refused_absent` o `refused_malformed`. Un cliente ofrece un certificado antes de
demostrar que posee la clave, así que esto cuenta intentos, no personas.

{{< callout type="error" >}}
**Por qué en modo cerrado ante el fallo**

Tome un host que confía en una segunda autoridad: su propia CA para el
personal, una CA de pruebas, la CA de preproducción de EFP añadida al mismo
archivo, o un cliente de go-authn/bridge sin concesiones. Los certificados
de esa autoridad no llevan concesión. Si una concesión ausente significara
«sin restricción», el filtro no filtraría nada en ese host.

Aquí, un certificado sin concesión se rechaza salvo que
`ssh_accept_ungranted` diga lo contrario. Una concesión presente que no se
puede analizar se rechaza diga lo que diga: una autoridad que la escribió
pretendía restringir el certificado. go-authn/sshcert describe paso a paso
el [escenario de varias CA](https://github.com/go-authn/sshcert#why-fail-closed-the-multi-ca-scenario).
{{< /callout >}}

### Con la CA de EFP {#with-efps-ca}

`trusted_user_ca_file` es el `TrustedUserCAKeys` de OpenSSH. Ponga en él la clave que
EFP publica en <https://sshca.my-eurohpc.eu/config>, como indica la
[página de confianza](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-trust/) de
EFP. Esa URL devuelve `{"PublicKey": "ssh-ed25519 ..."}`; el archivo contiene el
valor de `PublicKey` en una línea propia:

```sh
curl -s https://sshca.my-eurohpc.eu/config | jq -r '.PublicKey' > /etc/fileshare/efp-ssh-ca.pub
```

Después, establezca `ssh_domains` con los nombres que usa la concesión de su entidad
de alojamiento. El único principal de un certificado de EFP es el identificador de
MyAccessID (`<id>@myaccessid.org`), así que ese es el nombre de usuario con el que
inicia sesión. Un bloque `user` con ese nombre, sin credencial propia, lo recibe:

```hcl
user "u1234@myaccessid.org" {}
```

### Con go-authn/bridge {#with-go-authnbridge}

Un cliente de [go-authn/bridge](https://github.com/go-authn/bridge) (v0.19.0 o
posterior) con `ssh_certificates = true` y
`ssh_domain_grants = ["files.example.org"]` emite certificados concedidos para este
host. Con `ssh_principal_claim = "voperson_id"`, su único principal es el
`voperson_id` de la persona, como en el perfil de EFP, y ese es el bloque `user` que
hay que escribir aquí. Confíe en la CA de bridge como este servidor confía en
cualquier otra:

- como `trusted_user_ca_file`, para cuentas locales. bridge sirve su clave en el
  formato de EFP en `/ssh/config`, así que la línea `curl | jq` de arriba funciona
  con la URL de bridge;
- o como el `ssh_ca_file` del bloque `oidc`, para
  [las personas que el proveedor avala](#people-the-identity-provider-vouches-for).

Los demás clientes de bridge no escriben concesión. Si sus certificados deben seguir
funcionando aquí, para eso está `ssh_accept_ungranted`. Pero deja entrar **todos**
los certificados sin concesión de todas las autoridades de confianza, así que es
preferible dar una concesión a esos clientes.

### Un certificado vinculado a una dirección {#a-certificate-pinned-to-an-address}

El `ssh_source_address` de bridge escribe la opción crítica `source-address`, como
hace `ssh-keygen -O source-address=...`. Un certificado que la lleva se acepta
**solo desde esas direcciones**, con o sin `ssh_domains`. Un inicio de sesión desde
una dirección que el certificado permite se deja entrar, siempre que su concesión
nombre también este host cuando `ssh_domains` está establecido. Un inicio de sesión
desde cualquier otra dirección se rechaza.

| fileshare | `trusted_user_ca_file` | `ssh_ca_file` de `oidc`, y OpenPubkey |
|---|---|---|
| antes de la v0.22.0 | aplicada | **rechazado desde cualquier dirección** |
| v0.22.0, sin `ssh_domains` | aplicada | **rechazado desde cualquier dirección** |
| v0.22.0, con `ssh_domains` | **rechazado desde cualquier dirección** | **rechazado desde cualquier dirección** |
| v0.22.1, con o sin `ssh_domains` | aplicada | aplicada |

«Rechazado desde cualquier dirección» falla en modo cerrado: la restricción del
certificado nunca se relajó, solo se perdió el inicio de sesión. Antes de la v0.22.1,
el sshd de go-filesystems/sftp rechazaba cualquier opción crítica en los certificados
que pasaba a fileshare para que este decidiera; eso abarca los del proveedor, los de
OpenPubkey y, con `ssh_domains`, todos los certificados. La v0.22.1 se basa en
go-filesystems/sftp v0.5.1, cuyo sshd acepta ahí `source-address` y lo aplica en
todas las rutas de certificado, frente a la dirección de la que procede la conexión,
IPv4 o IPv6.

Cualquier otra opción crítica (`force-command`, `verify-required`, ...) se rechaza,
ya que este servidor no actúa en consecuencia.

## Las personas que el proveedor de identidad avala {#people-the-identity-provider-vouches-for}

Desde la v0.11.0, un bloque `oidc` llega también a SFTP; no con un token, que SSH no
tiene dónde poner, sino con un **certificado**:

```hcl
oidc {
  issuer           = "https://login.example.org"
  audience         = "fileshare"
  ssh_ca_file      = "/etc/fileshare/bridge-ca.pub"   # go-authn/bridge's ssh_ca
  opkssh_client_id = "opkssh"                         # OpenPubkey logins
  opkssh_max_age   = "24h"                            # 12h, 24h, 48h, 1week
}
```

- **Un certificado firmado por la CA SSH del proveedor**: `bridge ssh-cert` escribe
  uno tras un inicio de sesión a través de la federación. Su principal es la
  persona, su extensión `groups@go-authn.org` sus grupos, de modo que se aplican las
  [reglas `oidc:groups:`]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).
  **No** es `trusted_user_ca_file`, cuyos certificados se refieren a cuentas
  locales: el certificado de una autoridad local que reivindica los grupos del
  proveedor se interpreta como la cuenta local que nombra.
- **Un certificado OpenPubkey**, como el que escribe `opkssh login`: firmado por la
  propia clave del usuario, con un ID token que se compromete con esa clave. Se
  comprueba como lo comprueba `opkssh verify` —el verificador de
  [openpubkey](https://github.com/openpubkey/openpubkey) (la firma del proveedor
  frente a sus claves publicadas, el compromiso del nonce, el client ID, la
  antigüedad) y después la clave del certificado frente a la del token—, y el nombre
  de usuario SSH debe ser el nombre de usuario del token, porque aquí no hay archivo
  `auth_id` que haga corresponder el uno con el otro:
  `sftp alice@univ-example.fr@files.example.org`. `opkssh_client_id` debería ser un
  client ID propio, nunca el de otra aplicación: el ID token viaja a cada servidor en
  el que la persona inicia sesión.

Para ver los grupos que lleva un certificado:

```sh
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub     # the Extensions section
```

y el servidor indica, en cada inicio de sesión federado,
`sftp: alice@univ-a.fr, vouched for by the provider, in groups [...]`.

{{< callout type="error" >}}
**Avalado no significa admitido**

Alguien a quien el proveedor avala sigue siendo un desconocido aquí salvo
que una regla lo nombre, que `trust_all` indique que el proveedor es el
directorio, o que `local_names = true` indique que los nombres del proveedor
son los de este servidor y que una cuenta local tiene el suyo: la misma
prueba que supera un token por WebDAV. Un certificado del proveedor **sin
principal**, válido para *cualquiera* según la propia definición del
formato, se rechaza.
{{< /callout >}}

`-tags noopenpubkey` deja fuera el verificador de OpenPubkey (unos 2 MB); una
configuración con `opkssh_client_id` en una compilación así se rechaza.

## Retirar un certificado {#taking-a-certificate-back}

{{< callout type="error" >}}
**Un certificado se comprueba al iniciar sesión**

Revocar a la persona en el proveedor no revoca por sí solo un certificado ya
emitido: este abre SFTP hasta que caduca, y una sesión SFTP ya abierta sigue
abierta hasta entonces; estas personas no están en el directorio, así que
ninguna [recarga]({{< relref "/administration/reload.md" >}}) les concierne. El margen
es la vida útil del certificado: el `ssh_ca { validity }` de bridge (12h por
defecto, 168h como máximo, nunca más allá del final de la sesión del IdP) y
`opkssh_max_age` aquí. Manténgalo tan corto como permita el reinicio de
sesión de los clientes. Desde la v0.17.0, una sesión opkssh ya abierta
también termina en `opkssh_max_age`, diga lo que diga el certificado (que la
persona firma ella misma).
{{< /callout >}}

Dos mecanismos cierran ese margen:

- la **[KRL]({{< relref "/security/revocation-lists.md" >}})** del proveedor (`ssh_krl_url`):
  un certificado revocado se rechaza al iniciar sesión, y cada operación de una
  sesión que abrió vuelve a consultar la lista, incluidos los archivos abiertos;
- las **[señales compartidas]({{< relref "/security/shared-signals.md" >}})** (`ssf`): todo
  lo que el proveedor emitió a la persona antes de un `session-revoked` de CAEP se
  rechaza; es la única forma de alcanzar un certificado OpenPubkey, que no figura en
  ninguna lista.

Ambos fallan en modo cerrado.

## Sin contraseña {#no-password}

Un cliente que la pide hace justo lo que las claves existen para evitar.
