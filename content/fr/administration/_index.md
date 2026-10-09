---
title: "L'API d'administration"
linkTitle: "Administration"
weight: 30
description: "L'API d'administration gRPC qui crée, modifie, désactive et accorde des partages pendant que le serveur tourne, où elle écoute, et ce qu'elle refuse."
tags: [administration, api d'administration, grpc]
---

Depuis la v0.8.0, les partages peuvent changer pendant que le serveur tourne. Un bloc `admin`
active un service gRPC qui crée, modifie et met hors ligne des partages sans
redémarrage et sans modifier de fichier.

```hcl
admin {
  listen       = "unix:///run/fileshare/admin.sock"   # made 0600
  state_file   = "/var/lib/fileshare/shares.json"
  source_roots = ["/srv/images", "/data"]
}
```

Rien n'écoute tant que le bloc n'est pas écrit.

## Ce qu'elle fait {#what-it-does}

Le service est
[`fileshare.admin.v1.AdminService`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto) :

| | |
|---|---|
| `CreateShare`, `UpdateShare`, `DeleteShare` | définir un partage à partir d'une image, d'un répertoire ou (depuis la v0.21.0) d'un volume ; modifier `read_only` ou `protocols` (sa source ne peut pas changer : supprimez-le et créez-en un autre) |
| `DisableShare`, `EnableShare` | mettre un partage hors ligne, puis le rétablir |
| `Grant`, `Revoke` | donner à un sujet un accès en lecture ou en écriture, ou le lui retirer |
| `ListShares`, `GetShare` | tous les partages, avec ce qu'ils servent, sur quels protocoles, et pourquoi pas sur les autres |
| `ListUsers`, `ListGroups` | qui une autorisation peut nommer ; chaque utilisateur avec les protocoles auxquels ses identifiants peuvent répondre |
| `ReloadDirectory` | relire les personnes, maintenant — voir [relire l'annuaire]({{< relref "/administration/reload.md" >}}) |
| `GetServerInfo` | le nom, la version, l'heure de démarrage, la génération et les écouteurs |
| `ListParents`, `CreateVolume`, `ResizeVolume`, `SnapshotVolume`, `DeleteVolume`, `GetVolume`, `ListVolumes` | du stockage créé par un approvisionneur privilégié, et des partages servis à partir de celui-ci — voir [volumes]({{< relref "/administration/volumes.md" >}}) (depuis la v0.21.0) |

Une autorisation nomme un **utilisateur**, un **groupe** (`@group` dans un fichier de configuration), une
**valeur `oidc:groups:`** ou un **nom `oidc:user:`** — le vocabulaire qu'utilise le
fichier de configuration, voir
[les personnes que nomme le fournisseur d'identité]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).

`grpc.health.v1` répond sur le même écouteur. `reflection = true` dans le bloc
active la réflexion du serveur gRPC, pour `grpcurl`.

### Chaque modification dit ce qu'a fait sa mise en service {#every-change-says-what-serving-it-did}

Une modification répond par un message `Applied` :

```proto
message Applied {
  uint64 generation = 1;          // the generation now being served
  uint64 connections_closed = 2;  // how many the previous one had open, and closed
}
```

C'est pourquoi chaque RPC a son propre message de réponse au lieu de renvoyer
le partage lui-même : le `.proto` suit les règles de lint `STANDARD` de [buf](https://buf.build),
vérifiées par `buf lint` dans la CI de fileshare, plutôt que le « renvoyer la ressource »
de l'AIP-131 — c'est l'enveloppe qui porte `Applied`.

## Où elle écoute, et qui peut l'appeler {#where-it-listens-and-who-may-call-it}

{{< callout type="error" >}}
**En TCP, c'est TLS mutuel ou rien**

`tls_cert_file`, `tls_key_file` et `client_ca_file`, tous les trois,
**boucle locale comprise** : tout utilisateur local peut atteindre la boucle locale, et cette API
décide de qui lit les fichiers de qui. Un socket unix est créé en **0600**, et ses
permissions sont son contrôle d'accès.
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

L'écouteur est
[grpc-transports/control](https://github.com/grpc-transports/control). Chaque
modification est journalisée avec son auteur : le CN du certificat client, ou l'uid du pair
du socket.

## En HTTPS, pour des jetons OIDC {#over-https-for-oidc-tokens}

Depuis la v0.28.0, la même API — le même état, le même journal d'audit — est
aussi servie en HTTPS quand le bloc `admin` a un bloc `web` : **Connect,
gRPC-Web et gRPC sur un seul port d'écoute**, avec le certificat du
[bloc `tls`]({{< relref "/security/tls.md" >}}). C'est à elle que parlent les
interfaces web et natives.

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
    issuer "https://idp.partner.example" {
      audience = "fileshare-a.partner"
      subjects = ["5b0c…"]                # a person is (issuer, sub)
    }
  }
}
```

| champ | |
|---|---|
| `listen` | une adresse TCP ; refusé sans bloc `tls` — un jeton porteur vaut un mot de passe |
| `issuer "<url>"` | un bloc par fournisseur d'identité, son `iss` ; donné deux fois, il est refusé |
| `audience` | l'`aud` qu'un jeton doit nommer : **le nom de ce serveur** chez ce fournisseur ; obligatoire |
| `subjects`, `groups` | qui peut appeler : un `sub`, ou un groupe de la revendication de groupes ; au moins l'un des deux |
| `jwks_url` | se passe de la découverte OIDC, pour un fournisseur qui ne la publie pas |
| `groups_claim` | la revendication qui porte les groupes ; `groups` par défaut |

**Qui obtient une réponse.** Un appel est servi quand le vérificateur d'un
émetteur accepte son jeton porteur — signature, `iss`, un `aud` qui nomme ce
serveur, dates — **et** que le bloc de cet émetteur nomme le sujet du jeton ou
l'un de ses groupes. Un sujet inscrit chez un émetteur n'est personne chez un
autre : une personne, c'est (émetteur, `sub`). La vérification porte sur les
seuls en-têtes, avant qu'un octet du message soit décodé.

**Une audience par serveur.** Un jeton adressé à plusieurs serveurs pourrait
être rejoué par n'importe lequel d'entre eux auprès des autres (RFC 8707,
RFC 9068) : chaque serveur a donc la sienne, et une interface qui pilote
plusieurs serveurs demande un jeton par serveur.

**Ce qu'apprend un appelant refusé.** « Refusé », rien de plus. La raison, et
chaque modification faite, vont au journal d'audit sous `oidc=<émetteur> <sub>`.

**Pas de CORS.** Les navigateurs ne l'appellent pas directement : ils passent par
le serveur des interfaces, qui détient les jetons. Les requêtes sont bornées à
1 Mio et chaque phase d'une connexion a un délai. Chaque émetteur est contacté
**au démarrage**, pour lire ses clés ; un émetteur injoignable arrête le
démarrage, en le nommant.

## Ce qu'elle touchera, et ce qu'elle ne touchera pas {#what-it-will-and-will-not-touch}

**Elle gère les partages qu'elle a créés.** Un partage écrit dans la configuration est
listé, avec les mêmes champs, et une modification de sa définition est refusée avec
`FAILED_PRECONDITION` : un partage défini à deux endroits est une question à laquelle personne ne veut
répondre, et c'est le fichier qui définit celui-là. Les partages de l'API vivent dans
`state_file`, écrit de façon atomique, et sont de nouveau servis au démarrage suivant — c'est
pourquoi un bloc sans `state_file` est refusé.

**Une source doit se trouver sous `source_roots`**, une fois résolue — liens suivis, `..`
retirés — et c'est le chemin résolu qui est conservé. Sans `source_roots`, l'API
**ne peut créer aucun partage** : le processus peut lire `/dev` et `/etc`,
et personne n'a voulu les remettre à un appelant. C'est revérifié à chaque démarrage,
et un partage de `state_file` qui ne se trouve plus sous une racine arrête le démarrage,
en le nommant.

**Chaque partage a au moins une autorisation.** Un partage sans autorisation est ouvert à quiconque
s'authentifie. Le fichier peut le dire délibérément ; un appel d'API ne devrait pas le dire
par omission. `CreateShare` exige donc une autorisation, et révoquer la dernière est
refusé — supprimez plutôt le partage.

**Des noms qu'un protocole peut transporter.** Un nom de partage fait au plus 80 caractères, ne
commence ni ne finit par une espace, n'est ni `.` ni `..`, et ne contient aucun caractère
non imprimable ni aucun de `:*?"<>|{}%`. Les sujets ne contiennent aucun caractère non imprimable
non plus, et un partage accepte au plus 1000 autorisations — le rôle d'un groupe bien avant cela.
Chacun de ces cas était accepté avant la v0.14.0, écrit dans `state_file`, puis
fatal à chaque démarrage.

**Aucun partage ne peut contenir** la configuration, le fichier d'état, ni les secrets
qu'ils nomment : quiconque y écrit réécrirait qui peut faire quoi. Depuis la v0.17.0,
cela inclut `authorized_keys_file`, le `dsn_file` et le
`bind_password_file` d'un bloc `users`, le `ca_file` du bloc `ssf`, et une base sqlite que nomme
un DSN : quelqu'un capable d'ajouter sa clé à l'`authorized_keys` d'un autre
se connecterait sous son identité. **Un partage ne peut pas non plus en contenir un autre** (voir
[répertoires]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})).

**Une modification est vérifiée comme une configuration, ouverte, consignée, puis
servie.** Une modification que le serveur ne peut pas honorer — une image qui ne s'ouvre pas, un
nom que personne n'a dans l'annuaire — est refusée, en laissant tels quels ce qui était servi et ce qui
était écrit.

## Désactiver un partage {#disabling-a-share}

**Désactiver, c'est le `available = no` de Samba** : le partage reste défini et toute
tentative de connexion échoue ; ses connexions ouvertes sont fermées et son image ou
son répertoire est **relâché**, si bien que le fichier peut être remplacé pendant qu'il est hors ligne.
`EnableShare` le sert de nouveau, et est refusé quand sa source ne peut plus être
ouverte. `CreateShare` peut aussi créer un partage déjà désactivé.

Contrairement à toutes les autres modifications, elle s'applique aussi à un partage **de la configuration** —
mettre un partage hors ligne est une opération, pas une définition — et elle survit à un
redémarrage : elle est conservée dans le fichier d'état. [`fileshare check`]({{< relref "/configuration/check.md" >}})
liste ce qui est hors ligne.

## Une modification est une nouvelle génération {#a-change-is-a-new-generation}

{{< callout type="error" >}}
**Une modification redémarre les serveurs de protocoles, et ferme leurs connexions**

SMB vérifie qui peut se connecter une fois par connexion d'arborescence, et SFTP construit
l'arborescence d'une personne une fois par connexion : une session qui aurait survécu à une révocation
garderait l'accès qui vient d'être retiré. « Révoqué, sauf pour ceux qui étaient déjà
connectés » n'est pas une révocation.
{{< /callout >}}

Une modification n'est donc pas appliquée *aux* serveurs en cours d'exécution : ils sont remplacés par de
nouveaux construits à partir de la nouvelle liste — une **génération** — comme le ferait un redémarrage,
sans libérer les ports. Les bibliothèques ne permettaient pas de faire autrement :
go-filesystems/smb a `Share` et pas de `Unshare`, nfs a `Export` et pas de
`Unexport`.

Les ports restent liés : un client qui se connecte pendant une modification **attend** au lieu
d'être refusé. Les images dont le partage n'a pas changé gardent leur pilote, jamais
ouvert une seconde fois. Les clients se reconnectent ; c'est le prix, payé à chaque
modification — c'est pourquoi l'API applique une modification par appel plutôt qu'une par
champ.

## Limites {#limits}

- `--isolate` ne va pas encore avec un bloc `admin` — ni avec un bloc `metrics` :
  il n'y a aucun processus unique auquel une modification pourrait s'appliquer. La combinaison est
  refusée.
- Un binaire compilé avec `-tags nogrpc` n'a pas d'API d'administration, et une configuration avec
  un bloc `admin` est **refusée** plutôt que servie sans API. Cette étiquette fait gagner
  11,7 Mo ; voir [ne compiler que ce que l'on veut]({{< relref "/operations/build-tags.md#the-admin-api-and-nogrpc" >}}).

## Modifier le `.proto` {#changing-the-proto}

Le code Go sous `proto/` est généré et commité : `go install` n'a donc pas besoin de
`protoc`. Après avoir modifié un `.proto` — celui de l'API d'administration, ou le
`proto/fileshare/provision/v1/provision.proto` de l'approvisionneur — régénérez les deux avec les
versions qu'épingle la CI de fileshare (protoc 34.1, protoc-gen-go v1.36.12,
protoc-gen-go-grpc v1.6.2). La CI les régénère et échoue quand le code commité
diffère, ou quand du code généré a été laissé non commité :

```sh
protoc -I proto --go_out=. --go_opt=module=github.com/go-fileshare/fileshare \
  --go-grpc_out=. --go-grpc_opt=module=github.com/go-fileshare/fileshare \
  proto/fileshare/admin/v1/admin.proto proto/fileshare/provision/v1/provision.proto
```
