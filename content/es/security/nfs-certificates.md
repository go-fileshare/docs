---
title: "NFS, con identidades a partir de certificados"
weight: 30
description: "Servir recursos compartidos restringidos por NFS a personas nombradas por un certificado de cliente X.509, revocado mediante una CRL, y lo que eso no puede prometer en un cliente compartido."
tags: [seguridad, nfs, tls, revocación]
---

NFSv3 por sí solo no autentica a nadie, y por eso un recurso compartido que nombra
quién puede usarlo [no se exporta por NFS]({{< relref "/protocols/nfs.md" >}}), salvo que
algo permita distinguir a las personas. Un bloque `kerberos` es una respuesta. Desde
la v0.12.0, un certificado de cliente que **nombra a una persona** es otra:

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file; required
  crl_state_file = "/var/lib/fileshare/nfs.crl.state"     # the order, across a restart
}
```

## Lo que transporta el certificado {#what-the-certificate-carries}

Tras un inicio de sesión en un proveedor de identidad, go-authn/bridge emite un
**certificado de cliente X.509 de corta duración** que nombra a la persona tal como
lo lee el `rpc.tlsservd -u` de FreeBSD:

- la persona: el SubjectAltName **otherName `1.3.6.1.4.1.2238.1.1.1`**, un
  UTF8String `user@domain`
  ([draft-cel-nfsv4-rpc-tls-othername](https://www.ietf.org/archive/id/draft-cel-nfsv4-rpc-tls-othername-04.html)
  lo describe; sus OID aún no están asignados);
- sus grupos: un URI de SubjectAltName por grupo,
  `tag:go-authn.github.io,2026:group:<group>`, un **URI de etiqueta** (tag URI,
  RFC 4151: sin registro, pensado exactamente para esto). El `rpc.tlsservd` de
  FreeBSD lee el primer otherName de identidad y omite cualquier otra entrada SAN.

{{< callout type="info" >}}
**Por qué un URI de etiqueta, y no una extensión propia**

Se diseñó una, bajo un arco derivado de un UUID (`2.25.<128-bit number>`), y
se **midió**: `x509.ParseCertificate` de Go rechaza el certificado entero
—*malformed extension OID field*— porque allí un arco de identificador de
objeto ASN.1 es un `int`. Todo servidor TLS en Go habría rechazado el
certificado. Un SAN de tipo URI no necesita OID, y todos los analizadores lo
leen. Tampoco `urn:x-…`: el RFC 8141 suprimió los espacios de nombres URN
experimentales, y un analizador estricto los rechaza.
{{< /callout >}}

## Lo que se pregunta en cada llamada {#what-every-call-is-asked}

Con `identity = "certificate"`, un recurso compartido que nombra personas se sirve
por NFS **sin kerberos**. Cada llamada sobre él debe llegar por TLS, con un
certificado que:

1. **no está en la CRL**;
2. nombra **exactamente a una** persona;
3. nombra a alguien que este servidor admite **igual que admite un token**: una
   regla, `trust_all`, `domains`; consulte
   [las personas que nombra el proveedor de identidad]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}});
4. nombra a alguien **que el recurso compartido permite**.

Y, con un [bloque `ssf`]({{< relref "/security/shared-signals.md" >}}), el certificado debe
haberse emitido (su `NotBefore`) **después** de cualquier revocación de esa persona.

## Revocación: la CRL {#revocation-the-crl}

**Se exige una CRL**: `crl_url` o `crl_file`. El certificado de una persona que nada
puede revocar sobrevive a su baja durante toda su vida útil.

| campo | por defecto | |
|---|---|---|
| `crl_url` | | solo HTTPS |
| `crl_file` | | o una ruta absoluta; uno u otro |
| `crl_ca_file` | las del sistema | fija las autoridades del servidor HTTPS de la CRL |
| `crl_refresh` | `1m` | con qué frecuencia se descarga |
| `crl_max_age` | `1h` | qué antigüedad puede tener la última copia válida |
| `crl_state_file` | | dónde se guarda la última CRL verificada, para que un reinicio recuerde cuál es más reciente |

Se descarga y se conserva como la [KRL]({{< relref "/security/revocation-lists.md" >}}), y
**falla en modo cerrado** de la misma forma; una CRL que ha superado su propio
**`NextUpdate`** cuenta como desconocida, permita lo que permita `crl_max_age`. Una
CRL debe estar firmada por la CA de clientes: una CRL que cualquiera podría haber
escrito revoca lo que quiera y, peor aún, lo des-revoca. Con la CRL accesible, **la
siguiente llamada después de que cambie se rechaza**; la vida útil del certificado
solo limita una revocación cuando la CRL no es accesible.

Desde la v0.16.2, la firma se comprueba sobre los bytes en bruto **antes** de
analizar la CRL, mediante [go-authn/revocation](https://github.com/go-authn/revocation):
si se analizaba primero, una CRL grande firmada por cualquiera costaba memoria y
segundos antes de rechazarse. Lo que este servidor no puede leer como una lista
completa también se rechaza: una CRL delta, una extensión crítica o de entrada
desconocida, la ausencia de número de CRL, la ausencia de `NextUpdate`. Una CRL con
un número inferior al de la que se tiene, o con el mismo número emitida antes, se
rechaza como una [KRL]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}})
más antigua.

Un campo de CRL sin `identity = "certificate"` se rechaza, y también
`identity = "certificate"` sin `tls = true` y `client_ca_file`: la identidad está en
el certificado que presenta el cliente.

## Lo que no puede prometer {#what-it-cannot-promise}

{{< callout type="error" >}}
**El certificado de un cliente Linux pertenece a un MONTAJE, no a una persona**

El servidor ve el certificado de una **conexión**, y un cliente Linux asocia
uno a un montaje (`tlshd`, el número de serie del keyring en
`mount -o xprtsec=mtls,...`). **Todos los usuarios de ese montaje actúan
como la persona que nombra el certificado.** En una estación de trabajo que
usa una sola persona, es exactamente ella; en una máquina en la que inician
sesión varias personas, es quien montó: use `sec=krb5` allí.

Por eso el RFC 9289 por sí solo se niega a prometer la autenticación de
usuarios, y por eso el simple
[NFS sobre TLS]({{< relref "/security/tls.md#nfs-over-tls-proves-the-machine-not-the-person" >}})
sigue probando solo la máquina.
{{< /callout >}}

## Medido con un cliente Linux real {#measured-with-a-real-linux-client}

Núcleo 6.17, ktls-utils 0.9, en la CI de go-filesystems/nfs:

- **El `MNT` de MOUNT llega sin cifrar**, diga lo que diga `xprtsec=`: el cliente
  de montaje del núcleo no tiene TLS. Se responde, y toda llamada NFS hecha sin TLS
  se rechaza después; así, una persona a la que el recurso compartido no permite el
  acceso ve *acceso denegado* en el primer acceso y no en el `mount`. **Nada de un
  recurso compartido cruza sin cifrar.**
- `tlshd` comprueba el certificado de este servidor solo frente al almacén de
  confianza **del sistema** —ignora `x509.truststore`— y exige que el certificado y
  la clave del cliente **pertenezcan a root, con la clave en modo 600**. Si no, falla
  con *gnutls: Error in the certificate (-43)*, sin nombrar ni lo uno ni lo otro.
