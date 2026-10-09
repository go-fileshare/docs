---
title: "État"
weight: 60
description: "La version de fileshare que décrivent ces pages, ce que chaque version a ajouté, ce qui a été vérifié avec de vrais clients, et ce qui n'est pas encore implémenté."
tags: [état, versions]
---

Ces pages décrivent **fileshare v0.28.0**.

## Ce que chaque version a ajouté {#what-each-release-added}

| | |
|---|---|
| v0.7.0 | un partage restreint en NFS, avec un [bloc `kerberos`]({{< relref "/protocols/nfs.md#the-refusal-that-used-to-be-permanent" >}}) |
| v0.8.0 | l'[API d'administration]({{< relref "/administration/_index.md" >}}) en gRPC ; les [partages de répertoires]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}) ; [`/healthz`, `/readyz`, `/metrics`]({{< relref "/administration/health.md" >}}) |
| v0.9.0 | [TLS]({{< relref "/security/tls.md" >}}) pour WebDAV, S3 et NFS, depuis des fichiers ou par ACME ; **WebDAV avec mots de passe refusé en clair** |
| v0.10.0 | [relire l'annuaire]({{< relref "/administration/reload.md" >}}) : `reload = "5m"`, `SIGHUP`, `ReloadDirectory` |
| v0.11.0 | [SFTP pour les personnes dont le fournisseur d'identité se porte garant]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}) : son AC SSH, et opkssh |
| v0.12.0 | révocation : une [KRL]({{< relref "/security/revocation-lists.md" >}}) pour les certificats SSH du fournisseur, une CRL pour les [identités NFS issues de certificats]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.13.0 | [signaux partagés]({{< relref "/security/shared-signals.md" >}}) : un `session-revoked` CAEP annule tout ce que le fournisseur a émis auparavant, sur tous les protocoles |
| v0.14.0 | les correctifs de la revue de sécurité : [des noms de partage qu'un protocole peut transporter]({{< relref "/administration/_index.md#what-it-will-and-will-not-touch" >}}), aucun partage contenant la configuration ou ses secrets, `source_roots` revérifié au démarrage, SSF `max_age` ≥ 1m et `retain` ≥ 169h |
| v0.15.0 | une [KRL obtenue par `ssh_krl_url` doit être signée et datée]({{< relref "/security/revocation-lists.md#signed-and-dated-from-ssh_krl_url" >}}) |
| v0.16.0 | les listes de révocation [ne reculent jamais]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}}), même après un redémarrage, avec `ssh_krl_state_file` / `crl_state_file` ; les ancres de confiance et les fichiers de listes ne peuvent pas se trouver dans un partage |
| v0.16.1 | exige Go 1.26.6 : neuf avis de sécurité atteignables dans la bibliothèque standard de 1.26.4 |
| v0.16.2 | la [signature d'une CRL NFS est vérifiée avant son analyse]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}) |
| v0.16.3 | [personne n'est désigné par une adresse électronique non vérifiée]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}) |
| v0.16.4 | tests uniquement |
| v0.16.5 | tests uniquement ([#36](https://github.com/go-fileshare/fileshare/issues/36) : un test lisait une erreur de transport comme un succès) |
| v0.16.6 | le refus NFS nomme les deux issues : un bloc `kerberos`, ou [`identity = "certificate"`]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.16.7 | go-authn/oidc v0.2.2 : la taille d'une clé RSA est comptée en bits. Auparavant, une clé de 1024 bits complétée par des octets nuls validait un jeton WebDAV ou un PK Token opkssh |
| v0.17.0 | les correctifs d'un audit de sécurité : un partage ne peut pas en contenir un autre ; les fichiers d'accès (`authorized_keys_file`, `dsn_file`, …) ne peuvent pas se trouver dans un partage ; WebDAV ne devient jamais anonyme parce que la lecture d'un annuaire est revenue vide ; S3 respecte `protocols` ; une connexion qui ne s'est pas authentifiée en 30 s est fermée ; une session opkssh se termine à `opkssh_max_age` ; une session révoquée reste révoquée et est fermée |
| v0.17.1 | go-authn/oidc v0.2.4 (pas de jeu de clés derrière une redirection vers http), servercert v0.3.0 (le chemin du cache ACME vérifié comme le fait StrictModes de sshd), krl v0.5.0, revocation v0.3.0 |
| v0.17.2 | go-filesystems/s3 v0.3.0 : S3 ne dit plus à un appelant non authentifié quelles clés d'accès existent ; les URL présignées fonctionnent, durent une semaine au plus, et sont refusées avant leur propre date |
| v0.18.0 | la CI épingle Go 1.27.1 au lieu de `stable` |
| v0.18.1 | `fileshare check` avertit quand un bloc `oidc` n'a pas de `domains` : [le nom nu d'un fournisseur atteint le compte local de même nom]({{< relref "/configuration/identity.md" >}}) |
| v0.19.0 | go-authn/directory v0.11.0 : un bloc `users "ldap"` refuse une [liaison en clair vers une autre machine]({{< relref "/configuration/identity.md" >}}) ; les mots de passe sont comparés sous forme d'empreintes, en temps constant |
| v0.20.0 | [`local_names`]({{< relref "/configuration/identity.md" >}}) : les noms du fournisseur ne sont des noms locaux que lorsque le bloc `oidc` le dit (audit de sécurité F4) |
| v0.21.0 | [volumes]({{< relref "/administration/volumes.md" >}}) : un `fileshare provisioner` privilégié crée des jeux de données ZFS, des sous-volumes btrfs et des répertoires à quota de projet XFS/ext4, et l'API d'administration sert des partages à partir d'eux ; un partage de répertoire plein donne une même réponse sur tous les protocoles ; les `allowed_uids` du socket d'administration |
| v0.21.1 | un [partage plein]({{< relref "/administration/volumes.md#a-full-share" >}}) le dit en SMB (`STATUS_DISK_FULL`) et en NFS (`NFS3ERR_NOSPC` / `NFS3ERR_DQUOT`) |
| v0.22.0 | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}) : un certificat SSH n'est accepté que là où sa délégation de domaine (`ssh-domain-grant@core.aai.geant.org`, le profil de l'AC SSH EuroHPC) nomme cet hôte, en refusant en cas de doute ; `ssh_accept_ungranted` laisse entrer les certificats sans délégation |
| v0.22.1 | go-filesystems/sftp v0.5.1 : un certificat [épinglé à une adresse]({{< relref "/protocols/sftp.md#a-certificate-pinned-to-an-address" >}}) (`source-address`) est accepté depuis cette adresse sous `ssh_domains`, ainsi que sur les certificats du fournisseur et ceux d'OpenPubkey ; auparavant, il était refusé depuis toutes les adresses |
| v0.22.2 | tests et documentation uniquement : `source-address` est testé depuis `::1` et depuis une seconde adresse IPv4 (`127.0.0.2`), sous `ssh_domains`, et sur un certificat OpenPubkey ; sous Linux, avec `FILESHARE_REQUIRE_JUDGE`, un juge absent échoue au lieu d'être ignoré |
| v0.23.0 | un [volume btrfs]({{< relref "/administration/volumes.md#the-size-a-client-sees" >}}) a la taille de son quota sur NFS, SMB et WebDAV (go-filesystems/nfs v0.7.0, smb v0.5.0, webdav v0.3.0 donnent la taille à chaque requête) ; auparavant, btrfs affichait le système de fichiers entier. Un serveur dont les partages viennent de l'API d'administration ne s'arrête plus au démarrage quand il sert NFS sans avoir encore de partage |
| v0.24.0 | lectures plus rapides : un GET WebDAV d'un partage de répertoire ou de volume part en `sendfile(2)` en HTTP clair (pas sous TLS), NFS lit et écrit par 1 Mio (rtmax/wtmax), et les clients SMB lisent plus de 64 Kio par requête (`CAP_LARGE_MTU`). Sur le runner de CI, un fichier de 256 Mio : WebDAV 6,6 Go/s, SMB 0,84, NFS 1,47 (un client) |
| v0.24.1 | sécurité : une connexion NFS inactive ne garde plus les 3 Mio de son plus gros appel (24 connexions inactives retenaient 59 Mio) ; un en-tête de trame SMB ne fait plus allouer au serveur les 8 Mio qu'il annonce avant l'arrivée des octets ; une requête SMB ne peut plus payer un crédit pour 4 Gio (go-filesystems/nfs v0.8.1, smb v0.6.1 et v0.6.2) |
| v0.25.0 | SFTP répond à `limits@openssh.com` : le `sftp` d'OpenSSH lit jusqu'à 255 Kio (+25 %) ; une session tient au plus 1024 handles ouverts (go-filesystems/sftp v0.6.0) |
| v0.26.0 | copie côté serveur sur chaque protocole qui en a une : un `COPY` WebDAV, un `copy-data` SFTP (`sftp cp` d'OpenSSH) et un COPYCHUNK SMB (Explorateur, `copy_file_range` sous Linux) sont faits sur le serveur, par `copy_file_range(2)` entre fichiers de l'hôte — un reflink sur btrfs et XFS, mesuré en CI : 0 octet consommé pour une copie de 8 Mio. Sécurité : un `COPY` WebDAV et chaque `GET`/`HEAD` S3 ne lisent plus tout le fichier en mémoire ; S3 diffuse des objets de toute taille. Correction : le sendfile des `GET` WebDAV de la v0.24.0 n'avait lieu que sur les partages en lecture seule — une enveloppe le masquait sur ceux en écriture ; il a lieu sur les deux depuis la v0.26.0 |
| v0.27.0 | ACME : le certificat est demandé **au démarrage**, et non à la première connexion — nouvel essai après 1 min, puis doublé jusqu'à 1 h, pendant que le serveur continue de servir ; un client qui n'envoie **pas de SNI** (connexion par adresse IP) reçoit le certificat du premier domaine au lieu d'une poignée de main échouée (go-authn/servercert v0.5.0). [HARICA]({{< relref "/security/tls.md#harica-gant-tcs-which-account-and-caa" >}}) : seul un compte Enterprise **Admin** (OV) délivre sans défi, et le CAA doit autoriser `harica.gr` |
| v0.28.0 | l'API d'administration est aussi servie [en HTTPS pour des jetons OIDC]({{< relref "/administration/_index.md#over-https-for-oidc-tokens" >}}) — Connect, gRPC-Web et gRPC sur un seul port d'écoute, une audience par serveur, les appelants désignés par sujet ou par groupe pour chaque émetteur — première étape vers les interfaces web et natives |

{{< callout type="warning" >}}
**Passer à la v0.21.0**

Un **partage de répertoire plein** répond différemment. `EDQUOT` (un quota) et
`ENOSPC` sont désormais une seule erreur, « no space left on device (the share is full) »,
sur tous les partages de répertoires, pas seulement les volumes : un `EDQUOT` qui donnait
`500` en WebDAV et `NFS3ERR_IO` en NFS donne maintenant `507` et `NFS3ERR_NOSPC`, et le texte de SFTP
est cette phrase, sans le chemin. SMB répond toujours « accès refusé ». Voir
[un partage plein]({{< relref "/administration/volumes.md#a-full-share" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Passer à la v0.20.0**

L'`alice` du fournisseur n'obtient plus les partages de l'`alice` locale. Un site
dont les noms du fournisseur sont ses noms locaux ajoute `local_names = true` au
bloc `oidc` (et définit `domains`) ; le serveur journalise cette ligne pour chaque nom
qu'il refuse faute de ce réglage.
{{< /callout >}}

{{< callout type="warning" >}}
**Passer à la v0.19.0**

Un bloc `users "ldap"` qui pointe vers `ldap://` sur une autre machine sans
`start_tls = true` **ne démarre plus** : utilisez `ldaps://`, ou StartTLS.
{{< /callout >}}

{{< callout type="warning" >}}
**Passer à la v0.17.0**

Une configuration avec un partage dans un autre, deux partages sur une même source,
ou un fichier d'accès (`authorized_keys_file`, `dsn_file`, `bind_password_file`,
le `ca_file` de `ssf`) dans un partage **ne démarre plus** ; l'erreur nomme
le partage et le chemin.
{{< /callout >}}

{{< callout type="warning" >}}
**Passer au-delà de la v0.9.0**

Une configuration qui sert WebDAV sans TLS sur une adresse que d'autres machines peuvent
atteindre, avec quelqu'un à authentifier, **ne démarre plus**. Ajoutez `tls = true`
et un bloc `tls`, ou `plaintext = true` derrière un proxy qui termine TLS. Voir
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

## Vérifié {#verified}

Ce sont des choses qu'on a fait faire à un client que ce projet n'a pas écrit, et non
des affirmations sur le code.

- **macOS** monte un partage en SMB pendant que `curl` écrit dans la même image
  en WebDAV, et relit ce que WebDAV a écrit. L'écriture WebDAV de bob sur un partage
  qu'il ne peut que lire donne `403` ; un mauvais mot de passe donne `401`.
- **go-smb2** — un client que ce projet n'a pas écrit — mène vingt lectures
  concurrentes en SMB pendant que vingt autres passent par WebDAV, sous `-race`, en CI.
- Les règles d'accès sont vérifiées **à travers chaque protocole** plutôt que dans la seule
  configuration : alice écrit et bob non, en SMB comme en WebDAV,
  et le partage n'est même pas proposé à NFS.
- Les certificats SFTP sont vérifiés face au **propre client d'OpenSSH**, qui
  refuse aussi la même clé dès que son certificat est mis de côté.
- Les identités NFS issues de certificats ont été mesurées avec un **vrai client Linux**
  (noyau 6.17, ktls-utils 0.9) dans la CI de go-filesystems/nfs — c'est ainsi que
  [`MNT` en clair et les pièges de `tlshd`]({{< relref "/security/nfs-certificates.md#measured-with-a-real-linux-client" >}})
  ont été découverts.
- La forme non signée de la KRL est une mesure : OpenSSH 9.6 et 10.3 lisent une KRL
  signée et ignorent la signature (go-authn/krl, contre les deux).
- **La voie d'interopérabilité de go-authn/bridge** ([#12](https://github.com/go-authn/bridge/pull/12),
  [#17](https://github.com/go-authn/bridge/pull/17)) juge fileshare v0.13.0
  face au vrai bridge : la KRL, la CRL en NFS, les jetons WebDAV révoqués par
  signaux partagés, un IdP désactivé par portée, et opkssh par signaux partagés.

## Pas encore {#not-yet}

**Les écritures en S3.** `PUT` et `DELETE` répondent 403. La bibliothèque sait écrire, mais
un partage qu'une personne ne peut que lire doit refuser au même endroit que SFTP, et
ce n'est pas câblé. Refuser vaut mieux qu'un objet à moitié écrit.

**Un jeton OIDC en S3.** Les jetons fonctionnent en [WebDAV]({{< relref "/protocols/webdav.md" >}}) et
nulle part ailleurs. La voie habituelle pour S3 est STS `AssumeRoleWithWebIdentity`,
qui échange le jeton contre des identifiants temporaires avec lesquels le client signe
ensuite — un mécanisme différent de l'acceptation d'un jeton porteur, et non
implémenté.

**`--isolate` avec un bloc `admin` ou `metrics`.** Refusé : il n'y a aucun processus
unique auquel une modification par l'API pourrait s'appliquer. Voir
[un processus par protocole]({{< relref "/operations/isolation.md#not-with-an-admin-or-a-metrics-block-yet" >}}).

**Une personne par utilisateur sur un client NFS Linux partagé, à partir d'un certificat.** Un client
Linux attache le certificat à un montage : chaque utilisateur du montage est donc la
personne qu'il nomme. C'est une propriété du client, et la réponse y est
`sec=krb5`. Voir [ce qu'elle ne peut pas promettre]({{< relref "/security/nfs-certificates.md#what-it-cannot-promise" >}}).

[S3 lui-même]({{< relref "/protocols/s3.md" >}}) est livré : un partage est un bucket, servi sur
la même arborescence par utilisateur que celle de SFTP.

## Mesuré, et énoncé comme mesuré {#measured-and-stated-as-measured}

Les affirmations de ces pages qui relèvent de l'assurance ou de la mesure plutôt que
de la reproduction d'un défaut, et qui sont écrites ainsi dans le source aussi :

- Huit goroutines qui écrivent à travers un pilote fat32 non enveloppé sous `-race`
  ne produisent aucune course aujourd'hui. Le [verrou partagé]({{< relref "/operations/locking.md" >}}) est une assurance
  sur un contrat — `go-filesystems/interface` ne promet rien sur les appels concurrents
  par chemin — et non le correctif d'une panne observée.
- Les [tailles selon les étiquettes de compilation]({{< relref "/operations/build-tags.md" >}}) ont été mesurées, y compris les
  13,2 Mo que coûte à lui seul `hashicorp/go-plugin`, ce qui a tranché
  contre un cadre de greffons — et les 11,7 Mo que coûte gRPC maintenant que l'API d'administration
  l'apporte, ce qui explique l'existence de `nogrpc`.
- Une extension de certificat propre pour les groupes a été conçue et **mesurée**
  face à l'analyseur X.509 de Go, qui a refusé le certificat entier ; les groupes
  voyagent donc sous forme d'[URI tag]({{< relref "/security/nfs-certificates.md#what-the-certificate-carries" >}}).
