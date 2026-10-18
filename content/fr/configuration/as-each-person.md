---
title: "Des répertoires servis sous l'identité de chacun"
weight: 15
description: "identity = \"system\" : un partage de répertoire dont les fichiers sont ouverts par un worker qui tourne sous l'identité de la personne connectée ; le noyau applique les permissions de l'arborescence, et ce qu'elle crée lui appartient."
tags: [configuration, partages, identité, sécurité]
---

D'ordinaire, un seul processus fileshare sert tout le monde sous un seul
compte : fileshare décide qui peut faire quoi, et chaque fichier qu'il crée
appartient à ce compte. Un partage de répertoire peut au contraire être servi
**sous l'identité de chacun**. Les fichiers de la personne connectée sont
ouverts par un processus qui tourne avec son uid, son gid et ses groupes : le
noyau applique les permissions de l'arborescence, et ce qu'elle crée lui
appartient.

```hcl
identity {
  service_user = "fileshare"   # le compte sous lequel les protocoles sont servis
  min_uid      = 1000          # la valeur par défaut ; l'uid 0 est refusé quoi qu'on écrive
  min_gid      = 1000
  deny_uids    = [65534]
  idle         = "2m"          # combien de temps un worker inutilisé est gardé
}

share "home" {
  directory = "/srv/home"
  identity  = "system"
}
```

## Trois processus {#three-processes}

`fileshare serve` doit être **lancé en root**. Il ne reste root que pour
démarrer le worker de chaque personne. Tout ce qui fait face au réseau tourne
dans un second processus, sous `service_user`.

| processus | tourne sous | fait |
|---|---|---|
| le **lanceur** (`fileshare serve`) | root | ouvre les ports d'écoute, démarre le front, démarre les workers quand le front le demande. Il ne parle aucun protocole. |
| le **front** | `service_user`, avec les groupes de ce compte | parle les protocoles et authentifie, comme `serve` l'a toujours fait. Pour un tel partage, il demande au lanceur une **personne et un partage**, jamais un uid ni un chemin. |
| un **worker** | la personne | sert un partage au front, par une socket que le lanceur lui a transmise |

Un worker vérifie d'abord qu'il tourne exactement sous l'uid, le gid et les
groupes attendus, et que `setuid(0)` échoue. Alors seulement il ouvre
l'arborescence, par un `os.Root`. Un même worker sert toutes les connexions
de cette personne à ce partage, en SFTP comme en WebDAV, et s'arrête quand
rien ne l'a utilisé pendant `idle`. Quand le serveur s'arrête, les workers
s'arrêtent avec lui.

## Qui est une personne sur ce système {#who-a-person-is-on-this-system}

Le lanceur résout la personne **lui-même**, dans cet ordre :

1. **L'annuaire**, s'il porte `uidNumber` et `gidNumber` :
   - les `attributes = { uidNumber = "2001", gidNumber = "2001" }` d'un bloc `user` ;
   - une source LDAP avec `attributes = ["uidNumber", "gidNumber"]` ;
   - la requête `users_attributes` d'une source SQL.

   Les groupes supplémentaires sont les groupes qui listent la personne (les
   `members` d'un bloc `group`, `memberUid`) et ceux qu'elle nomme
   (`memberOf`), chacun par son `gidNumber`. Un groupe qui n'en a pas n'est
   pas un groupe système.
2. **`/etc/passwd` et `/etc/group`**, par le nom.

**Refusés** : l'uid 0, le gid 0, `(uid_t)-1`, un uid sous `min_uid`, un gid
primaire sous `min_gid`, et tout uid de `deny_uids`. Un groupe supplémentaire
sous `min_gid` est écarté, et le journal le dit. Une personne trouvée nulle
part est refusée elle aussi : il n'y a pas de compte de repli partagé. Une
personne dont le fournisseur d'identité se porte garant n'obtient un worker
qu'avec [`local_names`]({{< relref "/configuration/identity.md" >}}), qui
fait de son nom un nom local.

Un front compromis ne peut donc demander qu'une personne qui existe et qui est
autorisée, sur un partage qui existe. C'est toujours lui qui décide **qui**
s'est authentifié, et le lanceur ne peut pas le vérifier à sa place.

## Ce que voit un client {#what-a-client-sees}

| | SFTP | WebDAV |
|---|---|---|
| un worker est pris | à la connexion, pour toute sa durée | à chaque requête ; les verrous vivent aussi longtemps que le worker |
| le lanceur refuse la personne | le partage n'apparaît pas dans son arborescence (une arborescence vide est une connexion refusée) | `403` |
| aucun worker ne peut démarrer | de même | `503` |

La raison va au journal du serveur, jamais au client.

{{< callout type="warning" >}}
**Pas encore en SMB, NFS ni S3**

Ils refusent un tel partage, et [`check`]({{< relref "/configuration/check.md" >}})
le dit. SMB et NFS ont besoin, dans leurs bibliothèques, d'un système de
fichiers par session. NFS ne le fera jamais que sous Kerberos, car
`AUTH_UNIX` n'est que l'affirmation du client (RFC 5531 §14).
{{< /callout >}}

## Ce que cela demande au système {#what-it-asks-of-the-system}

- **Unix** : Linux, les BSD et macOS. Sous Windows, le partage est refusé.
- **Lisibles par `service_user`** : la configuration et chaque fichier qu'elle
  nomme (fichiers de mots de passe, clé d'hôte, certificats), car c'est le
  front qui les lit.
- **Exécutable par les personnes** : le binaire, car chaque worker l'exécute
  sous l'identité de sa personne.
- **Un redémarrage** pour ajouter ou modifier un tel partage. Le lanceur lit
  la configuration une fois, et refuse un partage qu'il ne connaît pas plutôt
  que de laisser le front le servir.
- `--isolate` ne va pas avec.

La conception, et la raison de chaque choix, sont dans
[docs/identity.md](https://github.com/go-fileshare/fileshare/blob/main/docs/identity.md).
