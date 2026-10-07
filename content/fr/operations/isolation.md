---
title: "Un processus par protocole"
weight: 20
description: "Faire tourner chaque protocole dans son propre processus avec --isolate, et les règles qui le rendent sûr."
tags: [exploitation, isolation]
---

```sh
fileshare --config /etc/fileshare.d --isolate
```

```
smb    on 0.0.0.0:445  — process 19810, serving public, photos and scratch
webdav on 0.0.0.0:8080 — process 19811, serving public and photos
nfs    on 0.0.0.0:2049 — process 19812, serving public
```

Le parent ouvre les écouteurs, puis se ré-exécute **lui-même** une fois par protocole,
en ne remettant à chaque enfant que les partages que ce protocole peut servir.

## Vérifié avec `lsof`, pas en lisant le code {#checked-with-lsof-not-by-reading-the-code}

```
smb     (pid 19810) has open: scratch.img photos.img
webdav  (pid 19811) has open: photos.img
nfs     (pid 19812) has open: photos.img
```

L'image accessible en écriture est ouverte dans **un seul processus**, et l'enfant WebDAV
ne l'ouvre jamais.

C'est ce que les [étiquettes de compilation]({{< relref "/operations/build-tags.md" >}}) ne peuvent pas donner : une panique ou un tas
épuisé dans un protocole n'abat que ce protocole, chaque enfant peut être confiné par
tout ce que le système d'exploitation propose, et un bogue dans un analyseur ne peut pas atteindre une
image que ce processus n'a jamais ouverte.

## Ports privilégiés {#privileged-ports}

Sous Unix, le parent passe l'écouteur déjà lié à l'enfant, si bien qu'**un port privilégié
fonctionne avec des enfants non privilégiés** — le parent se lie au port 445, l'enfant n'a jamais
besoin du privilège.

Windows n'a pas d'`ExtraFiles` : là, l'enfant se lie lui-même à l'adresse ; l'isolation
est la même, la moitié « port privilégié » ne l'est pas.

## La règle qui la rend honnête {#the-rule-that-makes-it-honest}

Un enfant ouvre l'image lui-même : une image servie *en écriture* par deux protocoles
serait donc deux pilotes sur un même fichier **sans aucun verrou entre eux** — exactement ce
que le [verrou partagé]({{< relref "/operations/locking.md" >}}) empêche au sein d'un seul processus.

C'est refusé, et le refus dit comment corriger :

```
scratch is writable over smb and webdav, and one process per protocol means
that many drivers writing one file with no lock between them. Say
`protocols = ["smb"]` on the share, or make it read_only, or do not isolate.
```

`protocols = [...]` sur un partage est utile en soi, pas seulement sous
`--isolate` : un partage déclaré SMB seul est un partage dont on ne parle jamais au
processus WebDAV. Un bloc `serve` qui finirait par ne rien porter est refusé lui aussi,
avant qu'aucune image ne soit ouverte, en nommant les partages qui lui ont été retirés et pourquoi.

## Pas encore avec un bloc `admin` ou `metrics` {#not-with-an-admin-or-a-metrics-block-yet}

`--isolate` associé à un bloc [`admin`]({{< relref "/administration/_index.md" >}}) ou
[`metrics`]({{< relref "/administration/health.md" >}}) est refusé : les enfants ouvrent
les images et le parent n'ouvre rien ; il n'y a donc aucun processus unique auquel une
modification par l'API pourrait s'appliquer, ni dont une sonde interrogerait la disponibilité.
