---
title: "TLS, et certificats par ACME"
weight: 10
description: "Servir WebDAV, S3 et NFS sur TLS avec un certificat issu de fichiers ou d'une AC ACME, et pourquoi WebDAV avec mots de passe est refusé en clair."
tags: [sécurité, tls, acme]
---

Depuis la v0.9.0, **WebDAV et S3 sont servis en HTTPS, et NFS en
RPC-with-TLS** (RFC 9289), avec `tls = true` sur leur bloc `serve` et un seul
bloc `tls` qui indique d'où vient le certificat.

## Depuis des fichiers {#from-files}

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

Les fichiers sont **rechargés quand ils changent** : un renouvellement écrit par un autre
outil est pris en compte sans redémarrage. Le certificat est celui de
[go-authn/servercert](https://github.com/go-authn/servercert).

## Depuis une AC ACME {#from-an-acme-ca}

Let's Encrypt par défaut ; ou une AC (autorité de certification) qui vous connaît par **external account
binding** (EAB), comme GÉANT TCS via HARICA :

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

| champ | |
|---|---|
| `directory_url` | le répertoire ACME de l'AC ; vide, c'est Let's Encrypt |
| `domains` | les noms à demander (obligatoire) |
| `cache_dir` | où le compte et les certificats sont conservés (obligatoire) |
| `email` | le contact du compte, facultatif |
| `eab_key_id`, `eab_hmac_key_file` | external account binding ; la clé HMAC est un secret, elle est donc lue depuis un fichier — en base64url, telle que l'AC la remet (le base64 standard est lu aussi) ; tout le reste est refusé |
| `http_challenge` | où répondre au défi http-01, par exemple `"0.0.0.0:80"` |

### La faisabilité d'ACME dépend de la façon dont l'AC vérifie le nom {#whether-acme-can-work-is-decided-by-how-the-ca-checks-the-name}

| défi | l'AC se connecte au | donc |
|---|---|---|
| tls-alpn-01 (RFC 8737) | port **443** | un protocole TLS doit être servi sur le 443 |
| http-01 (RFC 8555 §8.3) | port **80** | `http_challenge = "0.0.0.0:80"` y répond — ou l'adresse vers laquelle un port 80 est redirigé |
| aucun | rien | un compte d'AC dont le domaine est **prévalidé** ne demande aucun défi : les comptes EAB entreprise de HARICA pour GÉANT TCS — si bien qu'un serveur que **personne à l'extérieur ne peut joindre** obtient tout de même son certificat |

La dernière ligne est celle qui compte pour un serveur de fichiers à l'intérieur d'un réseau de campus :
avec un compte GÉANT TCS dont l'établissement a déjà validé le domaine auprès de
HARICA, aucun port n'a besoin d'être ouvert sur Internet.

Un certificat est demandé à la **première connexion TLS qui nomme l'hôte**
(SNI) ; un client qui se connecte par adresse IP n'en obtient aucun.

{{< callout type="info" >}}
**ALPN**

Avec ACME, la liste ALPN du serveur contient `acme-tls/1`, et un serveur Go
refuse un client dont la liste n'a rien en commun avec la sienne. `http/1.1` est donc
**ajouté** pour WebDAV et S3 — tous les navigateurs et clients WebDAV le proposent —
et `sunrpc` pour NFS, comme RFC 9289 §5.2 identifie RPC-with-TLS. Pas `h2` :
HTTP/2 n'est pas proposé sur ces écouteurs, et le proposer serait un mensonge.
`acme-tls/1` reste sur chaque écouteur : tls-alpn-01 reçoit donc une réponse partout où
l'AC arrive.
{{< /callout >}}

## WebDAV n'est pas servi en clair {#webdav-is-not-served-in-the-clear}

{{< callout type="error" >}}
**RUPTURE en v0.9.0 : WebDAV avec mots de passe sur une adresse joignable est refusé**

HTTP Basic, c'est le mot de passe, en base64, à chaque requête, et un jeton porteur
vaut un mot de passe. Une configuration qui sert WebDAV **sans TLS** sur une
adresse que d'autres machines peuvent atteindre, avec des personnes à authentifier, est donc
**refusée, et non signalée par un avertissement** — un avertissement défile, et le mot de passe, lui, ne
revient pas :

```
webdav on 0.0.0.0:8080 would carry passwords in the clear: serve it with
tls = true, or -- when TLS is terminated in front of it, by a proxy -- say
plaintext = true
```
{{< /callout >}}

Une configuration qui démarrait avant la v0.9.0 avec
`serve "webdav" { addr = "0.0.0.0:8080" }` ne démarre plus après. Deux
issues :

```hcl
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true            # with a tls block
}
```

ou, quand un proxy inverse termine TLS devant lui et transmet en clair :

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true       # TLS is terminated in front of this, on purpose
}
```

Les adresses de **boucle locale** (`127.0.0.1`, `::1`, `localhost`) et un serveur
**sans personne à authentifier** — ni `user`, ni `users`, ni bloc `oidc` — ne sont
pas concernés. Un bloc `serve` sans `addr` atterrit sur la boucle locale, et n'est pas concerné
non plus.

`plaintext` ne vaut que pour WebDAV, le protocole qui envoie un mot de passe, et
`plaintext` associé à `tls` est refusé : l'un ou l'autre.

## Ce que dit `check` {#what-check-says}

[`fileshare check`]({{< relref "/configuration/check.md" >}}) affiche, par protocole, ce qui est
chiffré et ce qui est en clair **délibérément** :

```
webdav: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem
nfs: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem; clients present one signed by /etc/fileshare/tls/clients.pem (the machine, not the person)
```

et, pour WebDAV derrière un proxy,

```
webdav: in the clear on 10.0.0.5:8080, on purpose (plaintext = true): TLS must be terminated in front of it
```

## Par protocole {#per-protocol}

| protocole | `tls = true` | |
|---|---|---|
| WebDAV | oui | HTTPS |
| S3 | oui | HTTPS |
| NFS | oui | RFC 9289 ; voir ci-dessous |
| SMB | **refusé** | SMB 3 chiffre avec ses propres clés |
| SFTP | **refusé** | SFTP, c'est SSH |

## NFS sur TLS prouve la machine, pas la personne {#nfs-over-tls-proves-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

`client_ca_file` oblige un client à présenter un certificat de cette autorité —
**quel hôte monte** — et RFC 9289 laisse l'authentification des utilisateurs telle qu'elle était :
l'uid à l'intérieur reste celui qu'affirme `AUTH_SYS`. Un partage qui nomme qui peut
l'utiliser est donc **toujours refusé en NFS** sans bloc `kerberos` — ou sans
[identités issues de certificats]({{< relref "/security/nfs-certificates.md" >}}), qui est un autre usage du
même certificat.

TLS est **proposé, pas exigé** : un client qui ne le demande jamais se voit toujours
servir les partages ouverts. `client_ca_file` sans `tls = true` est refusé — il
ne vérifierait rien — et `client_ca_file` sur tout autre protocole que NFS est
refusé aussi.

## Ce qui est refusé par ailleurs {#what-else-is-refused}

- `tls = true` sans bloc `tls` : rien n'indique d'où vient le certificat.
- Un bloc `tls` qu'aucun bloc `serve` n'utilise : rien ne s'en servirait.
- Un `http_challenge` qui n'est pas une adresse d'écoute.
