---
title: "NFS — nada, salvo Kerberos o un certificado"
linkTitle: "NFS: nada, salvo Kerberos o un certificado"
weight: 50
description: "NFSv3 no autentica a nadie por sí solo; cómo Kerberos o los certificados de cliente le permiten servir recursos compartidos restringidos, y lo que añade TLS."
tags: [protocolos, nfs, kerberos]
---

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

{{< callout type="error" >}}
**NFSv3 por sí solo no autentica a nadie**

`AUTH_UNIX` es una **afirmación**: el cliente dice «uid 501» y la red no
puede contradecirlo. Tampoco hay cifrado.
{{< /callout >}}

Por eso **un recurso compartido que nombra quién puede usarlo no se exporta por
NFS**. El rechazo se imprime al arrancar y en [`check`]({{< relref "/configuration/check.md" >}}),
con el motivo:

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

Un recurso compartido sin `allow` ni `writers` —cualquiera que se conecte, en lectura
y escritura— sí se exporta por NFS, porque ahí no hay ninguna regla que el protocolo
pueda dejar de respetar.

## El rechazo que antes era permanente {#the-refusal-that-used-to-be-permanent}

Un bloque `kerberos` permite a NFS distinguir a las personas, y un recurso compartido
restringido se sirve entonces por él como en cualquier otro sitio:

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5` transporta un principal que **prueba un ticket**, en lugar de un uid que
el cliente afirma.

{{< callout type="info" >}}
**Se compara el reino (realm), no solo el nombre antes de la `@`**

Dos reinos pueden tener cada uno un `alice`, y solo uno de ellos es el suyo.
{{< /callout >}}

## Sobre TLS: la máquina, no la persona {#over-tls-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

Desde la v0.9.0, NFS se sirve sobre RPC-with-TLS (RFC 9289) con `tls = true` y un
[bloque `tls`]({{< relref "/security/tls.md" >}}). Cifra; con `client_ca_file` obliga al cliente
a presentar un certificado de esa autoridad, que indica qué **máquina** está
montando. El uid de dentro sigue siendo el que afirma `AUTH_SYS`, así que un recurso
compartido restringido **sigue rechazándose** por esta vía. TLS se ofrece, no se
exige: a un cliente que nunca lo pide se le siguen sirviendo los recursos compartidos
abiertos.

## Una persona, a partir de su certificado {#a-person-from-their-certificate}

Desde la v0.12.0, un certificado puede nombrar a una **persona** —tal como lo emite
go-authn/bridge tras un inicio de sesión en un proveedor de identidad—, y entonces un
recurso compartido que nombra personas se sirve por NFS sin kerberos:

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file: one is required
}
```

{{< callout type="error" >}}
**El certificado de un cliente Linux pertenece a un montaje**

Todos los usuarios de ese montaje actúan como la persona que nombra el
certificado. Correcto en una estación de trabajo que usa una sola persona;
incorrecto en una máquina en la que inician sesión varias personas: use
`sec=krb5` allí.
{{< /callout >}}

Lo que transporta el certificado, la CRL que se exige y lo que se midió con un
cliente Linux real (`MNT` sin cifrar, las trampas de `tlshd`) se encuentran en
[NFS, con identidades a partir de certificados]({{< relref "/security/nfs-certificates.md" >}}).
