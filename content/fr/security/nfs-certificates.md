---
title: "NFS, avec des identités issues de certificats"
weight: 30
description: "Servir des partages restreints en NFS à des personnes nommées par un certificat client X.509, révoqué par une CRL, et ce que cela ne peut pas promettre sur un client partagé."
tags: [sécurité, nfs, tls, révocation]
---

NFSv3 à lui seul n'authentifie personne : c'est pourquoi un partage qui nomme qui peut
l'utiliser n'est [pas exporté en NFS]({{< relref "/protocols/nfs.md" >}}) — sauf si quelque chose sait
distinguer les personnes. Un bloc `kerberos` est une réponse. Depuis la v0.12.0, un certificat
client qui **nomme une personne** en est une autre :

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file; required
  crl_state_file = "/var/lib/fileshare/nfs.crl.state"     # the order, across a restart
}
```

## Ce que porte le certificat {#what-the-certificate-carries}

Après une connexion auprès d'un fournisseur d'identité, go-authn/bridge émet un **certificat client X.509
de courte durée** qui nomme la personne de la façon dont `rpc.tlsservd -u` de FreeBSD le
lit :

- la personne : l'**otherName `1.3.6.1.4.1.2238.1.1.1`** du SubjectAltName, une
  UTF8String `user@domain`
  ([draft-cel-nfsv4-rpc-tls-othername](https://www.ietf.org/archive/id/draft-cel-nfsv4-rpc-tls-othername-04.html)
  la décrit ; ses OID ne sont pas encore attribués) ;
- ses groupes : une URI SubjectAltName chacun,
  `tag:go-authn.github.io,2026:group:<group>` — une **URI tag** (RFC 4151 : sans
  enregistrement, faite précisément pour cela). `rpc.tlsservd` de FreeBSD lit le premier
  otherName d'identité et ignore toutes les autres entrées SAN.

{{< callout type="info" >}}
**Pourquoi une URI tag, et pas une extension propre**

Une extension a été conçue, sous un arc dérivé d'un UUID (`2.25.<128-bit number>`), et
**mesurée** : `x509.ParseCertificate` de Go refuse le certificat entier —
*malformed extension OID field* — parce qu'un arc d'identifiant d'objet ASN.1 y est
un `int`. Tous les serveurs TLS écrits en Go auraient rejeté le certificat.
Un SAN de type URI n'a besoin d'aucun OID, et tous les analyseurs le lisent. Pas `urn:x-…` non plus :
RFC 8141 a supprimé les espaces de noms URN expérimentaux, et un analyseur strict
en refuse un.
{{< /callout >}}

## Ce que l'on demande à chaque appel {#what-every-call-is-asked}

Avec `identity = "certificate"`, un partage qui nomme des personnes est servi en NFS
**sans kerberos**. Chaque appel sur ce partage doit arriver sur TLS, avec un certificat
qui :

1. **ne figure pas dans la CRL** ;
2. nomme **exactement une** personne ;
3. nomme quelqu'un que ce serveur admet **de la même façon qu'il admet un jeton** — une règle,
   `trust_all`, `domains`, voir
   [les personnes que nomme le fournisseur d'identité]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}) ;
4. nomme quelqu'un que **le partage autorise**.

Et, avec un [bloc `ssf`]({{< relref "/security/shared-signals.md" >}}), le certificat doit avoir été
émis (son `NotBefore`) **après** toute révocation de cette personne.

## Révocation : la CRL {#revocation-the-crl}

**Une CRL est obligatoire** — `crl_url` ou `crl_file`. Le certificat d'une personne que
rien ne peut révoquer survit à son retrait pendant toute sa durée de vie.

| champ | par défaut | |
|---|---|---|
| `crl_url` | | HTTPS uniquement |
| `crl_file` | | ou un chemin absolu — l'un ou l'autre |
| `crl_ca_file` | celles du système | épingle les autorités du serveur HTTPS de la CRL |
| `crl_refresh` | `1m` | la fréquence à laquelle elle est récupérée |
| `crl_max_age` | `1h` | l'âge maximal de la dernière bonne copie |
| `crl_state_file` | | où est conservée la dernière CRL vérifiée, pour qu'un redémarrage se souvienne de laquelle est la plus récente |

Elle est récupérée et conservée comme la [KRL]({{< relref "/security/revocation-lists.md" >}}), et **refuse en cas de doute**
de la même façon ; une CRL qui a dépassé son propre **`NextUpdate`** compte comme inconnue, quoi que
permette `crl_max_age`. Une CRL doit être signée par l'AC des clients — une CRL que n'importe qui
aurait pu écrire révoque ce qui lui plaît et, pire, le dé-révoque. Avec
la CRL joignable, **l'appel qui suit sa modification est refusé** ; la
durée de vie du certificat ne borne une révocation que lorsque la CRL est injoignable.

Depuis la v0.16.2, la signature est vérifiée sur les octets bruts **avant** que la CRL ne soit
analysée, par [go-authn/revocation](https://github.com/go-authn/revocation) :
analysée d'abord, une CRL volumineuse signée par n'importe qui coûtait de la mémoire et des secondes avant
d'être refusée. Ce que ce serveur ne peut pas lire comme une liste complète est refusé
aussi : une CRL delta, une extension critique ou une extension d'entrée inconnue, l'absence de numéro de
CRL, l'absence de `NextUpdate`. Une CRL dont le numéro est inférieur à celui qu'il détient, ou de
même numéro mais émise plus tôt, est refusée comme une
[KRL]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}}) plus ancienne.

Un champ de CRL sans `identity = "certificate"` est refusé, de même que
`identity = "certificate"` sans `tls = true` ni `client_ca_file` : l'identité
se trouve dans le certificat que présente le client.

## Ce qu'elle ne peut pas promettre {#what-it-cannot-promise}

{{< callout type="error" >}}
**Le certificat d'un client Linux appartient à un MONTAGE, pas à une personne**

Le serveur voit le certificat d'une **connexion**, et un client Linux en attache
un à un montage (`tlshd`, le numéro de série du trousseau dans
`mount -o xprtsec=mtls,...`). **Chaque utilisateur de ce montage agit en tant que la personne
que nomme le certificat.** Sur un poste de travail utilisé par une seule personne, c'est exactement
elle ; sur une machine où plusieurs personnes se connectent, c'est celle qui a monté — utilisez
`sec=krb5` dans ce cas.

C'est pourquoi RFC 9289 à elle seule refuse de promettre l'authentification des utilisateurs, et pourquoi
le simple [NFS sur TLS]({{< relref "/security/tls.md#nfs-over-tls-proves-the-machine-not-the-person" >}})
ne prouve toujours que la machine.
{{< /callout >}}

## Mesuré avec un vrai client Linux {#measured-with-a-real-linux-client}

Noyau 6.17, ktls-utils 0.9, dans la CI de go-filesystems/nfs :

- **Le `MNT` de MOUNT arrive en clair**, quoi que dise `xprtsec=` : le
  client de montage du noyau n'a pas de TLS. Il reçoit une réponse, et tout appel NFS fait
  sans TLS est ensuite refusé — si bien qu'une personne que le partage n'autorise pas voit
  *accès refusé* au premier accès plutôt qu'au `mount`. **Rien d'un
  partage ne traverse en clair.**
- `tlshd` vérifie le certificat de ce serveur face au magasin de confiance du **système**
  seulement — il ignore `x509.truststore` — et veut que le certificat et la clé du client
  **appartiennent à root, la clé en mode 600**. Sinon il échoue avec
  *gnutls: Error in the certificate (-43)*, sans nommer ni l'un ni l'autre.
