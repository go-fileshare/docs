---
title: "Volumes : le stockage que crée fileshare"
weight: 10
description: "Créer du stockage ZFS, btrfs, XFS et ext4 par l'approvisionneur privilégié de fileshare, et servir des partages à partir de celui-ci."
tags: [administration, volumes, zfs, btrfs, quotas]
---

Depuis la v0.21.0, l'[API d'administration]({{< relref "/administration/_index.md" >}}) peut **créer le stockage qu'un partage
sert** — un **jeu de données ZFS**, un **sous-volume btrfs**, un **répertoire XFS ou ext4
sous quota de projet** — dimensionné, attribué et suivi, et servir un partage à partir de celui-ci.
Linux seulement. Ceph n'est pas encore pris en charge.

Créer ce stockage exige `CAP_SYS_ADMIN` (quotactl, les ioctl de quota btrfs,
`/dev/zfs`, mount(2)). `fileshare serve` analyse cinq protocoles réseau pour le compte
d'inconnus et ne détient aucun privilège : la partie qui en a besoin est donc un second rôle du
même binaire, `fileshare provisioner` (l'approvisionneur, provisioner), derrière un socket unix et un petit protocole
fermé :

```
                         admin gRPC (unix socket 0600 + peer uid, or mTLS)
  operator ─────────────────────────────────────────────▶  fileshare serve
                                                             (unprivileged)
                                                                  │
                     provision gRPC (unix socket, SO_PEERCRED = fileshare's uid)
                                                                  ▼
                                                    fileshare provisioner
                                                    (CAP_SYS_ADMIN, CAP_CHOWN, CAP_FOWNER)
                                                                  │
                                    go-fsctl: zfs · btrfs · projquota (ioctls, no CLI)
```

Un binaire plutôt que deux démons : une seule version à déployer, une seule définition de
protocole. Le stockage lui-même est géré par [go-fsctl](https://go-fsctl.github.io/)
(pur Go, sans commande `zfs` ni `btrfs`). La conception, et ce qui a changé en cours
de construction, se trouvent dans le
[`docs/volumes.md`](https://github.com/go-fileshare/fileshare/blob/v0.26.0/docs/volumes.md) du dépôt fileshare.

## Les deux processus, configurés {#both-processes-configured}

`fileshare serve`, en tant qu'utilisateur `fileshare` (uid 990) :

```hcl
# /etc/fileshare/fileshare.hcl
admin {
  listen          = "unix:///run/fileshare/admin.sock"
  state_file      = "/var/lib/fileshare/shares.json"
  source_roots    = ["/srv/fileshare/volumes"]
  provisioner     = "unix:///run/fileshare-provisioner/provisioner.sock"
  provisioner_uid = 0          # who must answer on that socket; 0 is the default
  allowed_uids    = [990]      # who may call this API
}
```

| clé | |
|---|---|
| `provisioner` | le socket de l'approvisionneur, `unix:///path`. Sans lui, tout appel sur les volumes répond `FAILED_PRECONDITION` et aucun partage n'est créé à partir d'un volume |
| `provisioner_uid` | l'uid sous lequel doit tourner le processus qui répond sur ce socket, lu depuis le noyau (`SO_PEERCRED`) à chaque connexion : un socket lié par quelqu'un d'autre n'obtient rien. Non défini, il vaut 0 |
| `allowed_uids` | facultatif, socket unix seulement : les seuls uid dont les appels reçoivent une réponse ; tout autre reçoit `PERMISSION_DENIED`, et est audité. Il ne peut que **restreindre** ce que permet le mode 0600 du socket (son propriétaire et root) — root, par exemple, peut alors ne plus appeler. Refusé en TCP, où c'est le certificat client qui fait la vérification |

Le chemin d'un volume doit se trouver sous `source_roots`, comme toute autre source.

`fileshare provisioner`, en tant que root :

```hcl
# /etc/fileshare-provisioner.hcl
provisioner {
  listen     = "unix:///run/fileshare-provisioner/provisioner.sock"
  client_uid = 990
  group      = "fileshare"
  max_volume = "10T"
  state_file = "/var/lib/fileshare-provisioner/volumes.json"

  parent "tank" {
    zfs  = "tank/fileshare"
    root = "/srv/fileshare/volumes/tank"
  }
  parent "fast" { btrfs = "/srv/fileshare/volumes/fast" }
  parent "plain" {
    xfs         = "/srv/fileshare/volumes/plain"
    project_ids = "100000-199999"
  }
}
```

| clé | |
|---|---|
| `listen` | `unix://<absolute path>` et rien d'autre. Le socket est créé en 0660, appartenant à l'utilisateur de l'approvisionneur (root) et à `<group>`, et doit se trouver dans un répertoire où personne d'autre que l'approvisionneur ne peut écrire — le **sien** (`/run/fileshare-provisioner`), pas celui de fileshare |
| `client_uid` | le seul uid auquel il répond : celui de fileshare. **0 est refusé** |
| `group` | possède la racine de chaque volume (`root:<group>`, mode 2770). fileshare doit en faire partie : il écrit dans un volume par le groupe, jamais en tant que propriétaire — le propriétaire d'un répertoire peut effacer son identifiant de projet sans privilège, et sortir ainsi d'un quota XFS/ext4 |
| `max_volume` | le plus grand quota qu'un volume peut avoir (`"10T"`, `"500G"`, `"1048576"`) ; au-delà, c'est `OUT_OF_RANGE` |
| `state_file` | ce que ce processus a créé — identifiants de sous-volumes btrfs, identifiants de projets XFS/ext4 — pour qu'après un redémarrage il sache ce qui lui appartient |
| `parent "<id>"` | où des volumes peuvent être créés, sous l'identifiant que nomme une requête. Exactement un parmi `zfs` (le jeu de données dont les volumes sont les enfants, avec `root` là où ils sont montés), `btrfs`, `xfs` ou `ext4` (le répertoire dans lequel ils sont créés) |
| `project_ids` | XFS/ext4 : la plage d'identifiants de projet que ce parent peut attribuer |
| `enable_quota` | btrfs : laisser l'approvisionneur activer les quotas lorsqu'ils sont désactivés. Sans cela, un parent btrfs aux quotas désactivés arrête le démarrage — les qgroups ralentissent chaque commit à mesure que les instantanés se multiplient, et c'est à l'opérateur de choisir |

Le fichier de l'approvisionneur ne contient rien d'autre que son bloc `provisioner` : le propre
fichier de fileshare qui lui serait remis par erreur est refusé plutôt qu'à moitié lu.

Un volume est toujours `<parent root>/<name>` : le jeu de données `tank/fileshare/<name>`
monté sur `/srv/fileshare/volumes/tank/<name>` (avec `refquota`), un sous-volume btrfs
limité par son qgroup, un répertoire XFS ou ext4 avec un identifiant de projet
pris dans la plage et une limite stricte de blocs. XFS et ext4 exigent l'option de montage `prjquota`
(ext4 aussi les fonctionnalités `project,quota`).

## La paire systemd {#the-systemd-pair}

L'approvisionneur :

```ini
# fileshare-provisioner.service
[Service]
ExecStart=/usr/local/bin/fileshare provisioner -c /etc/fileshare-provisioner.hcl
RuntimeDirectory=fileshare-provisioner
RuntimeDirectoryMode=0755
StateDirectory=fileshare-provisioner
StateDirectoryMode=0700
CapabilityBoundingSet=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
# Run as root, or as a dedicated user with:
# AmbientCapabilities=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
NoNewPrivileges=yes
SystemCallFilter=@system-service @mount quotactl quotactl_fd
SystemCallArchitectures=native
LockPersonality=yes
RestrictRealtime=yes
# NOT ProtectSystem= nor ReadWritePaths=: they give the service its own
# mount namespace, and the ZFS volumes it mounts would be seen by nobody
# else. And no Landlock: a Landlock-restricted thread cannot mount(2).
```

Le serveur, sans aucune capacité :

```ini
# fileshare.service
[Unit]
Requires=fileshare-provisioner.service
After=fileshare-provisioner.service

[Service]
User=fileshare
Group=fileshare
ExecStart=/usr/local/bin/fileshare serve -c /etc/fileshare/fileshare.hcl
RuntimeDirectory=fileshare
StateDirectory=fileshare
# No capability at all: CAP_SYS_RESOURCE would let it past an ext4 quota,
# and fileshare refuses to serve volumes with it.
CapabilityBoundingSet=
AmbientCapabilities=
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/srv/fileshare/volumes /var/lib/fileshare
```

`ProtectSystem=` convient au serveur : c'est l'approvisionneur qui monte, et
les montages se propagent vers l'espace de noms du serveur, pas dans l'autre sens : un volume ZFS
monté après le démarrage du serveur lui est donc visible.

{{< callout type="info" >}}
**Des exemples, pas des unités testées**

Aucune des deux unités n'est exercée en CI : le test de bout en bout lance les deux processus
sous `sudo`.
{{< /callout >}}

{{< callout type="warning" >}}
**Mettez d'abord à jour l'approvisionneur**

L'approvisionneur refuse un message portant un champ que sa version ne
définit pas — il ferme la connexion — si bien que `fileshare serve` et
`fileshare provisioner` doivent tourner dans la même version.
{{< /callout >}}

## Les appels de l'API d'administration sur les volumes {#the-admin-apis-volume-calls}

`fileshare serve` les relaie à l'approvisionneur
([`admin.proto`](https://github.com/go-fileshare/fileshare/blob/v0.26.0/proto/fileshare/admin/v1/admin.proto)) :

| | |
|---|---|
| `ListParents` | où des volumes peuvent être créés : l'identifiant de chaque parent, son type, sa racine, son espace libre et total, s'il prend des instantanés ; et `max_volume_bytes` |
| `CreateVolume` | un volume de `quota_bytes` (0 est refusé). Idempotent par nom : le même quota répond avec le volume existant (`created` à false), un autre quota donne `ALREADY_EXISTS` |
| `ResizeVolume` | modifier le quota, dans un sens comme dans l'autre — jamais en dessous de ce qui est utilisé (`FAILED_PRECONDITION`) |
| `SnapshotVolume` | un instantané nommé, en lecture seule : ZFS et btrfs ; `UNIMPLEMENTED` sur XFS et ext4 |
| `GetVolume`, `ListVolumes` | chaque volume avec son quota, ce qu'il utilise, ses instantanés et **les partages qui l'utilisent** |
| `DeleteVolume` | refusé tant qu'un partage l'utilise — supprimez d'abord le partage. Un volume qui contient des données ou des instantanés exige `destroy_data` |
| `CreateShare` avec `volume { parent, name }` | un partage servi à partir du volume, comme un répertoire |

`DeleteShare` ne supprime jamais un volume. « Les partages qui l'utilisent » sont les partages
créés à partir de lui **et** tout partage — y compris ceux des fichiers de configuration, servis,
désactivés ou indisponibles — dont l'image ou le répertoire se trouve à l'intérieur.

Un nom correspond à `^[a-z0-9][a-z0-9_-]{0,62}$`, et une requête ne porte jamais de
chemin : elle nomme un `parent` de la propre configuration de l'approvisionneur.

Avec `reflection = true` dans le bloc `admin`, `grpcurl` peut faire les appels.
Le socket est en 0600 et `allowed_uids` ci-dessus ne nomme que l'uid de fileshare : lancez-le
donc sous cet utilisateur :

```sh
S=/run/fileshare/admin.sock
A=fileshare.admin.v1.AdminService

sudo -u fileshare grpcurl -plaintext -unix $S $A/ListParents

# a 32 GiB volume on the XFS parent
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "34359738368"}' \
  $A/CreateVolume

# a share served from it
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"name": "projects",
       "volume": {"parent": "plain", "name": "projects"},
       "grants": [{"subject": {"group": "staff"}, "access": "ACCESS_WRITE"}]}' \
  $A/CreateShare

# grow it to 64 GiB
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "68719476736"}' \
  $A/ResizeVolume
```

### Codes de statut {#status-codes}

Les codes de l'approvisionneur reviennent tels qu'il les a donnés : `INVALID_ARGUMENT` (un nom
hors de la grammaire, un parent inconnu, un quota de 0), `OUT_OF_RANGE` (au-delà de
`max_volume`), `RESOURCE_EXHAUSTED` (le quota, ou l'augmentation d'un redimensionnement, dépasse
ce que le parent a de libre en ce moment ; ou la plage d'identifiants de projet est épuisée),
`ALREADY_EXISTS`, `NOT_FOUND`, `FAILED_PRECONDITION`, `UNIMPLEMENTED`.

Trois sont propres à fileshare :

- aucun `provisioner` configuré : `FAILED_PRECONDITION` ;
- l'approvisionneur refuse l'uid de ce serveur : `FAILED_PRECONDITION` aussi — c'est
  une erreur de déploiement (son `client_uid` est faux), pas quelque chose que l'appelant
  d'administration n'a pas le droit de faire ;
- l'approvisionneur ne répond pas : `UNAVAILABLE`.

## Ce qui est vérifié avant qu'un volume ne soit servi {#what-is-checked-before-a-volume-is-served}

fileshare demande à l'approvisionneur où se trouve le volume, puis vérifie, lui-même :

- que le chemin est propre, absolu et — une fois résolu — sous `source_roots` ;
- que c'est un répertoire ;
- que `statfs` signale le système de fichiers du type (`ZFS_SUPER_MAGIC`,
  `BTRFS_SUPER_MAGIC`, `XFS_SUPER_MAGIC`, `EXT4_SUPER_MAGIC`) ;
- **ZFS** : que c'est un point de montage — son périphérique n'est pas celui de son parent. Le répertoire d'un
  jeu de données non monté accepterait des écritures hors de tout quota. (Le jeu de données qui y est
  monté n'est pas vérifié : la réponse de l'approvisionneur ne le dit pas.)
- **btrfs** : que c'est la racine d'un sous-volume — un simple répertoire sous le parent n'a
  pas de qgroup ;
- **XFS et ext4** : qu'il porte un identifiant de projet non nul dont héritent ses nouveaux fichiers
  — un répertoire hors de tout projet est servi sans limite.

Le fichier d'état conserve le **nom** du volume, pas son chemin, et chaque démarrage redemande
et revérifie. Un volume disparu, qui échoue à une vérification, ou dont l'approvisionneur
ne répond pas en 10 s laisse son partage **défini et non servi** — c'est signalé
au démarrage, par [`fileshare check`]({{< relref "/configuration/check.md" >}}), et dans le
champ `unavailable` du partage — pendant que le reste du serveur démarre.
`EnableShare` le retente.

{{< callout type="error" >}}
**`fileshare serve` refuse de servir des volumes en tant que root ou avec `CAP_SYS_RESOURCE`**

ext4 permet à l'un comme à l'autre d'écrire au-delà d'un quota de projet (fs/quota/dquot.c,
`ignore_hardlimit`) : root a écrit 16 Mio dans un projet de 8 Mio dans la CI de
go-fsctl/projquota. XFS n'a pas cette exemption, mais la règle est une seule
règle pour le processus, sur tous les types : fileshare lit son uid effectif et
`CapEff`/`CapPrm` dans `/proc/self/status` (une capacité permise est à un
`capset(2)` de devenir effective), et un statut qu'il ne peut pas lire compte comme
la détenant. Il refuse de **servir**, pas les appels sur les volumes — créer du stockage
en tant que root est sans danger, y écrire en tant que root ne l'est pas. `fileshare check` dit
ce qu'il en est. D'où le refus de `client_uid = 0` par l'approvisionneur, et le
`CapabilityBoundingSet=` vide de l'unité du serveur ci-dessus.
{{< /callout >}}

## Un partage plein {#a-full-share}

XFS signale un projet plein avec `ENOSPC` ; ext4, btrfs et ZFS avec `EDQUOT`. Depuis la
v0.21.1, tous les protocoles répondent « plein » :

| protocole | plus d'espace (`ENOSPC`) | un quota (`EDQUOT`) |
|---|---|---|
| WebDAV | `507 Insufficient Storage` | `507 Insufficient Storage` |
| NFS | `NFS3ERR_NOSPC` (28) | `NFS3ERR_DQUOT` (69), comme RFC 1813 et le nfsd de Linux |
| SFTP | `SSH_FX_FAILURE`, « no space left on device (the share is full) » — la version 3 n'a pas de code pour cela | la même chose |
| SMB | `STATUS_DISK_FULL` (0xC000007F) | `STATUS_DISK_FULL`, comme le fait Samba (« Windows apps need this, not NT_STATUS_QUOTA_EXCEEDED ») |
| S3 | servi en lecture seule | servi en lecture seule |

En v0.21.0, SMB répondait `STATUS_ACCESS_DENIED` et NFS répondait à un quota par
`NFS3ERR_NOSPC`.

Cela s'applique à tout [partage de répertoire]({{< relref "/configuration/shares.md" >}}), pas seulement
aux volumes — voir la [note de mise à jour]({{< relref "/status.md" >}}).

## La taille que voit un client {#the-size-a-client-sees}

Chaque protocole demande au serveur la taille d'un partage à chaque requête : `FSSTAT`
pour NFS, `FileFsFullSizeInformation` pour SMB et les propriétés de quota de la RFC 4331
pour WebDAV.

- **ZFS, XFS et ext4** : la taille est ce que dit `statfs` à l'intérieur du volume, auquel
  le noyau répond par le quota (`refquota` pour ZFS, le quota de projet pour XFS et ext4).
- **btrfs** (depuis fileshare v0.23.0) : le `statfs` de btrfs ignore les qgroups et
  annonce le système de fichiers entier, donc le serveur prend les chiffres de
  l'approvisionneur. La taille est le quota du volume, et l'espace libre est le quota
  moins les octets référencés du qgroup, ou l'espace libre du système de fichiers s'il
  est plus petit. Le serveur interroge l'approvisionneur en arrière-plan, au plus une
  fois toutes les 5 secondes tant que des clients demandent, et répond entre-temps avec
  les derniers chiffres. btrfs met à jour le compte d'un qgroup quand il valide une
  transaction (toutes les 30 secondes par défaut) : l'espace libre vu par un client peut
  donc avoir ce retard sur les écritures. Avant la v0.23.0, un volume btrfs affichait le
  système de fichiers entier ; le quota s'appliquait quand même.

L'espace utilisé vient de l'approvisionneur (`used_bytes`) : lire l'utilisation d'un projet XFS ou ext4
exige `CAP_SYS_ADMIN`, que fileshare n'a pas.

## Ce que promet l'approvisionneur {#what-the-provisioner-promises}

- **Un seul uid, lu depuis le noyau.** Le pair est identifié par `SO_PEERCRED`
  à sa connexion, et quiconque n'est pas `client_uid` — root compris — reçoit
  `PERMISSION_DENIED` avant qu'un seul octet de la requête ne soit décodé.
- **Un ensemble fermé de verbes.** Une méthode hors de son service, ou un message qui
  ne se décode pas ou porte un champ qu'il ne définit pas, ne reçoit aucune réponse : la
  connexion est fermée.
- **Il ne touche jamais à ce qu'il n'a pas créé.** ZFS : la propriété `fileshare:volume`
  propre au jeu de données — posée sur le jeu de données, puisque les propriétés utilisateur sont
  héritées et qu'une marque héritée n'est pas la sienne. btrfs : l'identifiant et
  l'uuid du sous-volume consignés dans `state_file`. XFS/ext4 : un identifiant de projet dans la
  plage du parent *et* consigné. Vérifié avant chaque redimensionnement, instantané et suppression.
- **La suppression refuse les données et les instantanés** sauf si `destroy_data` est défini, et est
  journalisée avant d'avoir lieu.

## Mesuré {#measured}

Dans la CI de fileshare, en tant que root sur de vrais pools et des systèmes de fichiers adossés à des périphériques loop, avec `fileshare serve` tournant en tant que `nobody` et piloté par
l'API d'administration : les écritures en WebDAV et en SFTP s'arrêtent au quota d'un volume de 32 Mio sur
les quatre types.

## Pas encore {#not-yet}

- **Ceph.** Les quotas CephFS et RBD viendront dans une phase ultérieure.
