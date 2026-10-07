---
title: "La configuration"
linkTitle: "Configuration"
weight: 10
description: "Comment la configuration HCL est organisée, ce que porte un bloc user, et ce qui est refusé au démarrage."
tags: [configuration]
---

Un seul fichier, ou un répertoire de petits fichiers. `--config /etc/fileshare.d` les
fusionne : un utilisateur dans un fichier et un partage dans un autre vont donc ensemble.

```hcl
name = "ATTIC"

user "alice" { password_file = "/etc/fileshare/alice.pw" }
user "bob"   { password_file = "/etc/fileshare/bob.pw" }

group "family" { members = ["alice", "bob"] }

share "photos" {
  image   = "/srv/photos.img"
  allow   = ["@family"]        # a group, or a person, in either list
  writers = ["alice"]          # bob gets it read-only
}

share "scratch" {
  image = "/srv/scratch.img"   # anyone who authenticates, read-write
}

tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true                  # HTTP Basic is the password: never in the clear
}
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

{{< callout type="error" >}}
**Depuis la v0.9.0, WebDAV avec mots de passe n'est pas servi en clair**

`serve "webdav" { addr = "0.0.0.0:8080" }` — l'exemple que montrait cette page
jusque-là — est désormais **refusé au démarrage** dès que quelqu'un s'authentifie :
HTTP Basic, c'est le mot de passe à chaque requête. Indiquez `tls = true` avec un bloc
`tls`, comme ci-dessus, ou `plaintext = true` quand un proxy termine TLS devant
lui. Voir [TLS]({{< relref "/security/tls.md" >}}).
{{< /callout >}}

Un partage peut aussi être un [périphérique]({{< relref "/configuration/shares.md#a-device-not-only-an-image" >}}) ou un
[répertoire de l'hôte]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}) plutôt qu'une
image.

## Ce que peut porter un bloc `user` {#what-a-user-block-can-carry}

Tous les exemples ci-dessus utilisent `password_file`, le cas courant, qui fut
un temps le seul que ce programme lisait. Ce n'est pas tout le bloc, et
la différence décide **sur quels protocoles une personne peut être servie** — elle mérite
donc d'être réunie en un seul endroit plutôt que découverte lors d'un montage.

| champ | ce que c'est | ce que cela permet |
|---|---|---|
| `password` | un mot de passe, dans le fichier | tous les protocoles auxquels un mot de passe répond |
| `password_file` | la même chose, dans un fichier à part | la même chose, sans le secret dans la configuration |
| `nt_hash` | `MD4(UTF16LE(password))`, 32 caractères hexadécimaux | **SMB**, pour une personne dont ce site ne détient pas le mot de passe |
| `authorized_keys` | des lignes `authorized_keys`, en ligne | **SFTP** |
| `authorized_keys_file` | la même chose, depuis un fichier | **SFTP** |
| `totp_secret` | un secret de code à usage unique en base32 | rien pour l'instant : il est lu et vérifié, et aucun protocole ici ne demande de second facteur |

⛔ `password` et `password_file` ensemble sont refusés au démarrage, avec le nom de la
personne : deux réponses à « quel est son mot de passe » posent la question de savoir laquelle
l'emporte, et une configuration ne devrait pas avoir à être lue deux fois pour le savoir. De même pour
`authorized_keys` et `authorized_keys_file` ensemble.

⛔ Un bloc `user` qui n'a ni `password`, ni `password_file`, ni `authorized_keys`,
ni `authorized_keys_file` est refusé lui aussi — *has no way to authenticate* —
sauf si `trusted_user_ca_file` est défini. `nt_hash` et `totp_secret` ne comptent pas
ici.

{{< callout type="default" >}}
**Un utilisateur défini en ligne PEUT être servi en SMB**

Jusqu'à récemment, ce bloc ne lisait que les champs de mot de passe : servir SMB
à quelqu'un inscrit ici revenait à confier à ce fichier son mot de passe en
clair — ou à le déplacer dans une base de données. Avec `nt_hash`, ce n'est plus le cas :
un site qui stocke ce que stocke Samba peut écrire cela à la place. Le bloc a toujours
besoin, à côté, de l'un des champs ci-dessus — des clés, par exemple — ou de
`trusted_user_ca_file` : `nt_hash` seul est refusé au démarrage. Voir
[ce qu'une source peut prouver]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}),
qui est le même tableau, une couche plus haut.
{{< /callout >}}

Un groupe s'écrit `@name` partout où une personne pourrait figurer.

## Ce qui est refusé au démarrage, plutôt que plus tard {#what-is-refused-at-startup-rather-than-later}

{{< callout type="warning" >}}
**Un nom qui n'appartient à personne**

`allow = ["alise"]` exclurait sinon Alice de son propre partage et
démarrerait sans broncher. Un nom qu'aucun bloc `user` ni aucun [annuaire `users`]({{< relref "/configuration/identity.md" >}})
ne connaît est refusé, tout comme un `@group` qu'aucun d'eux n'a.
{{< /callout >}}

{{< callout type="warning" >}}
**Un partage qui nomme qui peut l'utiliser, en NFS**

Une configuration qui dit *photos appartient à alice* et un protocole qui remet
photos à quiconque se connecte ne peuvent pas être honorés tous les deux. Un tel partage n'est pas
exporté en NFS — sauf si un bloc `kerberos` ou `identity = "certificate"`
permet à NFS de distinguer les personnes — et un bloc serve `nfs` qui n'a plus rien à
porter est refusé. Voir [NFS]({{< relref "/protocols/nfs.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Un fournisseur qui n'a nulle part où donner sa parole**

Un bloc [`oidc`]({{< relref "/protocols/webdav.md" >}}) est refusé sauf si WebDAV est
servi — le seul protocole qui porte un en-tête `Authorization` — ou si SFTP
est servi avec les certificats du fournisseur activés (`ssh_ca_file` ou
`opkssh_client_id`, voir [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})).
SMB et NFS n'ont de place ni pour l'un ni pour l'autre.
{{< /callout >}}

{{< callout type="warning" >}}
**Des mots de passe en clair**

WebDAV sur une adresse que d'autres machines peuvent atteindre, sans `tls = true` ni
`plaintext = true`, avec des personnes à authentifier. Voir
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Un bloc qui ne peut pas être servi en sécurité**

Un bloc `admin` sans `state_file`, ou qui écoute en TCP sans TLS
mutuel ; un bloc `tls` qu'aucun bloc `serve` n'utilise ; un `reload` plus court qu'une
seconde ; une liste de révocation récupérée en HTTP simple. Chacun est décrit
là où il a sa place : [administration]({{< relref "/administration/_index.md" >}}),
[TLS]({{< relref "/security/tls.md" >}}), [rechargement]({{< relref "/administration/reload.md" >}}),
[révocation]({{< relref "/security/_index.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Un protocole avec lequel ce binaire n'a pas été compilé**

On le dit *tel quel*, plutôt que « ce protocole n'existe pas » — la différence
entre une faute de frappe et une [étiquette de compilation]({{< relref "/operations/build-tags.md" >}}).
{{< /callout >}}

Un bloc `serve` sans `addr` écoute sur la boucle locale, `127.0.0.1`, sur un port
qui ne demande aucun privilège : 4445 pour SMB, 8080 pour WebDAV, 2222 pour SFTP, 2049
pour NFS, 9000 pour S3.

## `protocols` sur un partage {#protocols-on-a-share}

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

Indique quels protocoles le portent. Utile en soi — un partage déclaré SMB seul est
un partage dont on ne parle jamais au processus WebDAV — et obligatoire dans certaines
configurations sous [`--isolate`]({{< relref "/operations/isolation.md" >}}). Un bloc `serve`
qui finirait par ne rien porter est refusé lui aussi, avant qu'aucune image ne soit ouverte,
en nommant les partages qui lui ont été retirés et pourquoi — sauf si un bloc `admin` est
là pour lui donner des partages plus tard.
