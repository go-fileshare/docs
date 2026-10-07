---
title: "NFS — rien, sauf Kerberos ou un certificat"
linkTitle: "NFS : rien, sauf Kerberos ou un certificat"
weight: 50
description: "NFSv3 n'authentifie personne à lui seul ; comment Kerberos ou des certificats clients lui permettent de servir des partages restreints, et ce qu'apporte TLS."
tags: [protocoles, nfs, kerberos]
---

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

{{< callout type="error" >}}
**NFSv3 à lui seul n'authentifie personne**

`AUTH_UNIX` est une **affirmation** : le client dit « uid 501 » et le réseau ne peut pas
le contredire. Il n'y a pas non plus de chiffrement.
{{< /callout >}}

Donc **un partage qui nomme qui peut l'utiliser n'est pas exporté en NFS**. Le refus
est affiché au démarrage et dans [`check`]({{< relref "/configuration/check.md" >}}), avec sa
raison :

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

Un partage sans `allow` ni `writers` — quiconque se connecte, en lecture-écriture — est
exporté en NFS, car il n'y a là aucune règle que le protocole pourrait manquer
d'honorer.

## Le refus qui était autrefois définitif {#the-refusal-that-used-to-be-permanent}

Un bloc `kerberos` permet à NFS de distinguer les personnes, et un partage restreint
est alors servi par ce biais comme partout ailleurs :

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5` transporte un principal qu'un **ticket prouve**, plutôt qu'un uid que le
client affirme.

{{< callout type="info" >}}
**Le royaume est comparé, pas seulement le nom avant le `@`**

Deux royaumes peuvent avoir chacun une `alice`, et un seul d'entre eux est le vôtre.
{{< /callout >}}

## Sur TLS : la machine, pas la personne {#over-tls-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

Depuis la v0.9.0, NFS est servi en RPC-with-TLS (RFC 9289) avec `tls = true` et un
[bloc `tls`]({{< relref "/security/tls.md" >}}). Il chiffre ; avec `client_ca_file`, il oblige un
client à présenter un certificat de cette autorité — c'est-à-dire quel **hôte** monte.
L'uid à l'intérieur reste celui qu'affirme `AUTH_SYS` : un partage restreint est donc
**toujours refusé** par ce biais. TLS est proposé, pas exigé : un client qui ne le
demande jamais se voit toujours servir les partages ouverts.

## Une personne, d'après son certificat {#a-person-from-their-certificate}

Depuis la v0.12.0, un certificat peut nommer une **personne** — tel que go-authn/bridge en émet
un après une connexion auprès d'un fournisseur d'identité — et un partage qui nomme des personnes est alors
servi en NFS sans kerberos :

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file: one is required
}
```

{{< callout type="error" >}}
**Le certificat d'un client Linux appartient à un montage**

Chaque utilisateur de ce montage agit en tant que la personne que nomme le certificat. Juste sur
un poste de travail utilisé par une seule personne ; faux sur une machine où plusieurs personnes se connectent —
utilisez `sec=krb5` dans ce cas.
{{< /callout >}}

Ce que transporte le certificat, la CRL exigée, et ce qui a été mesuré
avec un vrai client Linux (`MNT` en clair, les pièges de `tlshd`) se trouvent dans
[NFS, avec des identités issues de certificats]({{< relref "/security/nfs-certificates.md" >}}).
