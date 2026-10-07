---
title: "Une image, plusieurs protocoles, un verrou"
weight: 30
description: "Pourquoi une image servie par plusieurs protocoles est enveloppée dans un seul verrou partagé, et ce que cette enveloppe conserve."
tags: [exploitation, verrouillage]
---

Chaque serveur de cette famille sérialise lui-même les appels au pilote, parce que
[`go-filesystems/interface`](https://github.com/go-filesystems/interface)
ne promet rien sur les appels concurrents par chemin — et chacun d'eux détient
**son propre** verrou, sans rien savoir des autres.

Servir une image par trois protocoles à la fois n'aurait donc aucun verrou commun
nulle part. L'image est donc enveloppée **une seule fois**, ici, et la même enveloppe est
remise à tous.

## Énoncé tel que mesuré, pour ne pas l'exagérer {#stated-as-measured-so-as-not-to-overstate-it}

Huit goroutines qui écrivent à travers un pilote fat32 **non enveloppé** sous `-race`
ne produisent **aucune course aujourd'hui**.

C'est une assurance sur un contrat, pas la reproduction d'un défaut — ext4 et ntfs
ne sont pas fat32, et une mise à jour de pilote n'est pas une chose que ce programme devrait avoir à
réauditer.

## L'enveloppe laisse passer les capacités {#the-wrapper-carries-capabilities-through}

Elle ne les masque pas :

- un pilote qui répond à `Opener` reçoit en retour un **`File` verrouillé** ;
- celui dont le `File` répond à `WritableFile` conserve ses **écritures positionnelles**.

L'écart entre celles-ci et le repli sur le fichier entier a été mesuré à
**~70×** ailleurs dans la famille : une enveloppe qui les effacerait serait un
défaut de performance déguisé en sécurité.

## Sous `--isolate`, le verrou ne peut rien {#under-isolate-the-lock-cannot-help}

Un processus enfant ouvre l'image lui-même : il n'y a donc pas d'enveloppe partagée entre
les enfants. C'est pourquoi une image servie en écriture par deux protocoles est
[refusée]({{< relref "/operations/isolation.md#the-rule-that-makes-it-honest" >}}) plutôt que servie
discrètement sans verrou.
