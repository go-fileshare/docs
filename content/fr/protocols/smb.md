---
title: "SMB — NTLMv2"
linkTitle: "SMB : NTLMv2"
weight: 10
description: "SMB avec NTLMv2 : ce qu'un annuaire doit détenir, le port privilégié sous isolation, et ce qu'un rechargement de l'annuaire fait aux connexions ouvertes."
tags: [protocoles, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

Le mot de passe **ne traverse jamais le réseau** : un client envoie une preuve calculée à partir
de celui-ci. Le partage dit à un lecteur qu'il n'est que lecteur — dans le masque d'accès, avant qu'il
n'essaie, plutôt qu'en faisant échouer une écriture plus tard.

## Ce qu'un annuaire doit détenir {#what-a-directory-must-hold}

NTLMv2 a besoin du mot de passe lui-même ou de son MD4 (`sambaNTPassword`, une colonne
`nt_hash`). **Rien d'autre ne convient.**

Un annuaire qui ne fait que *vérifier* les mots de passe — une liaison LDAP, une colonne bcrypt —
ne peut pas répondre à SMB, aussi bonne que soit la vérification, car le serveur doit calculer
la même preuve que le client. C'est une propriété du protocole, pas une
limite de ce programme.

[`check`]({{< relref "/configuration/check.md" >}}) affiche un `-` dans la colonne SMB pour une telle
personne, au moment de la configuration, plutôt que de la laisser le découvrir lors d'un
montage.

## Kerberos : un ticket au lieu d'un mot de passe {#kerberos}

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"   # nfs/fs.example.org et cifs/fs.example.org
}

serve "smb" {
  addr = "0.0.0.0:445"
  ntlm = false          # Kerberos seul : aucun mot de passe n'est jamais vérifié
}
```

Le bloc `kerberos` qui permet à NFS de distinguer les personnes permet aussi à SMB
d'accepter un ticket. **Un seul keytab** détient les deux services, chaque protocole
répondant pour le sien : SMB ne lit que les principaux `cifs/<hôte>` du royaume du
bloc, donc un ticket émis pour `nfs/` y est refusé. Un keytab sans principal `cifs/`
laisse SMB tel qu'il était, NTLM seul, et le serveur le dit au démarrage.

Le principal passe par la même règle que pour NFS (l'`alice` d'un autre royaume n'est
personne) et doit ensuite être **quelqu'un que cette configuration connaît** : un
ticket prouve que le KDC connaît la personne, pas que ce serveur la connaît. Les
listes `allow` et `writers` du partage s'appliquent comme pour un mot de passe. Une
session prend fin avec son ticket, et un client Windows s'authentifie de nouveau tout
seul.

Depuis fileshare v0.50.0, le **keytab** doit être un fichier ordinaire, appartenant à
root ou au compte sous lequel tourne fileshare, ni lisible ni modifiable par les
autres, ni modifiable par son groupe : `root:fileshare` `0640` et
`fileshare:fileshare` `0600` sont acceptés, `krb5:fileshare` `0640` est refusé. Cela
vaut pour SMB et NFS. Le nom et le royaume du client viennent du **ticket**, jamais
de l'authentificateur que le client envoie aussi (jcmturner/gokrb5#577, RFC 4120
section 3.2.3) ; NFS exige go-authn/krb5 v0.3.4 ou plus récent, ce que livre v0.50.0.

`ntlm = false` est refusé au chargement sans bloc `kerberos`, et le serveur refuse de
démarrer si le keytab ne contient aucune clé `cifs/` pour le royaume : un serveur où
personne ne peut se connecter n'en est pas un. Avec le bloc, une personne dont
l'annuaire ne détient aucun hachage NT peut quand même utiliser SMB, et
[`check`]({{< relref "/configuration/check.md" >}}) le dit.

Une machine Windows **hors domaine** l'atteint avec un ticket de ce royaume :
`ksetup /addkdc` et `ksetup /addhosttorealmmap` une fois, un ticket d'octroi de tickets
dans la session de connexion, puis `net use \\fs.example.org\photos` — par le nom,
car Windows se rabat sur NTLM pour une adresse. Mesuré avec Windows 11 25H2 (non
joint) : le client propose MS-KRB5, Kerberos, NEGOEX et NTLMSSP, envoie un AP-REQ
Kerberos, monte le partage, liste, lit et écrit avec la signature exigée, et
n'envoie aucun message NTLM.

## SMB sur QUIC {#quic}

```hcl
tls {
  cert_file = "/var/lib/caddy/certificates/acme-v02.api.letsencrypt.org-directory/files.example.org/files.example.org.crt"
  key_file  = "/var/lib/caddy/certificates/acme-v02.api.letsencrypt.org-directory/files.example.org/files.example.org.key"
}

serve "smb" {
  addr      = ":443"          # UDP : sans conflit avec un proxy HTTPS sur TCP 443
  transport = "quic"
  tls       = true
  # client_cert    = "required"
  # client_ca_file = "/etc/fileshare/client-ca.pem"
}
```

SMB 3.1.1 sur QUIC est le même protocole dans TLS 1.3 sur UDP 443, avec l'ALPN
`smb` : `net use \\files.example.org\share /TRANSPORT:QUIC` sous Windows 11, TCP
445 fermé. Le certificat est celui du bloc `tls`, ses fichiers sont rechargés
quand ils changent : les pointer vers le stockage ACME d'un proxy inverse suit
ses renouvellements. Un client Windows compare le nom du serveur aux noms DNS
du certificat : composez un nom, jamais une adresse (une adresse force aussi
NTLM). Un proxy sur la même machine ne doit pas offrir HTTP/3 sur ce port UDP
(Caddy : `protocols h1 h2`).

Par défaut, seul Kerberos distingue les personnes, avec un principal
`cifs/files.example.org` dans la keytab. `client_cert = "required"` ajoute un
certificat client chaîné à `client_ca_file`, exigé dans la poignée de main TLS
avant tout octet SMB : il dit quelle machine peut se connecter, le principal
Kerberos reste la personne. Le fichier de CA est lu au démarrage. Sur QUIC le
serveur accorde un crédit par requête et n'annonce aucun large MTU
([MS-SMB2] 3.3.5.4) : une lecture ou écriture fait au plus 64 Kio.
`identity = "system"` et les enfants isolés ne servent pas encore QUIC et le
serveur refuse de démarrer avec eux. Un bloc `serve "smb"` porte un seul
transport : lancez une seconde instance pour TCP.

## Un port privilégié sans serveur privilégié {#a-privileged-port-without-a-privileged-server}

Sous [`--isolate`]({{< relref "/operations/isolation.md" >}}) sous Unix, le parent se lie au port 445
et passe l'écouteur à l'enfant : le processus qui parle réellement SMB
n'a donc jamais besoin du privilège. Windows n'a pas d'`ExtraFiles` : là, l'enfant
se lie lui-même à l'adresse ; l'isolation est la même, la moitié « port privilégié »
ne l'est pas.

## Vérifié face à un client que ce projet n'a pas écrit {#verified-against-a-client-this-project-did-not-write}

[go-smb2](https://github.com/cloudsoda/go-smb2) mène vingt lectures concurrentes
en SMB pendant que vingt autres passent par WebDAV, sous `-race`, en CI. macOS
monte un partage en SMB pendant que `curl` écrit dans la même image en WebDAV, et
relit ce que WebDAV a écrit.

## Quand les personnes changent {#when-the-people-change}

SMB fixe qui peut se connecter à un partage **au moment où le partage est ajouté**, et le vérifie
une fois par connexion d'arborescence (tree connect). Une nouvelle personne trouvée par un [rechargement de l'annuaire]({{< relref "/administration/reload.md" >}})
est donc ajoutée sur place, serveur SMB en cours d'exécution compris, sans toucher aucune connexion ;
mais tout ce qui est retiré — une personne, un identifiant, les listes développées d'un partage —
constitue une nouvelle génération, et les connexions SMB ouvertes sont fermées : une
révocation atteint ainsi les sessions déjà ouvertes. Un partage dont le groupe `allow` s'est
vidé **n'est pas du tout proposé en SMB** : son `AllowUsers` vide se lirait comme
« tout le monde ».

## Requêtes en vol {#requests-in-flight}

Depuis go-filesystems/smb v0.8.1, une connexion garde jusqu'à 32 requêtes en vol, et le serveur jusqu'à 128 sur l'ensemble des connexions. Les réponses sont associées par `MessageId`. Une trame contenant un seul `READ` d'un fichier ordinaire s'exécute à côté d'autres ; les chaînes composées (traitées en bloc, dans l'ordre), les requêtes asynchrones et `WRITE` s'exécutent seules. Une trame dispose de 30 s pour arriver une fois commencée. Le mécanisme est le module partagé go-filesystems/inflight, que SFTP et NFS utilisent aussi.
