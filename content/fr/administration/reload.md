---
title: "Relire l'annuaire"
weight: 30
description: "Comment et quand le serveur relit ses annuaires d'utilisateurs, et ce qu'un rechargement fait aux partages et aux connexions ouvertes."
tags: [administration, identité, rechargement]
---

```hcl
reload = "5m"   # and on SIGHUP, and on the admin API's ReloadDirectory
```

Les personnes viennent d'annuaires que d'autres choses modifient — go-authn/bridge écrit
un mot de passe d'application dans une table, et supprime la ligne quand la personne est
désactivée. Sans rechargement, le serveur sert ceux qui étaient là à son démarrage.
Depuis la v0.10.0, il les relit :

- à chaque intervalle `reload`, une durée Go d'**au moins une seconde** — un annuaire
  lu plusieurs fois par seconde est une charge pour la base de données de quelqu'un, pas un serveur
  plus à jour ;
- sur **`SIGHUP`** (Unix) ;
- sur le **`ReloadDirectory`** de l'[API d'administration]({{< relref "/administration/_index.md" >}}), qui répond en indiquant qui
  a été ajouté, supprimé et modifié, si une nouvelle génération a démarré, et des remarques qu'une
  personne devrait lire — un groupe qui n'existe plus, quelqu'un qu'un partage nomme
  et qui n'est plus dans l'annuaire.

Sans `reload`, l'annuaire n'est lu qu'au démarrage, sur `SIGHUP` et sur
`ReloadDirectory`.

Ce sont les blocs `users` — SQL, LDAP — qui sont relus. Les blocs `user` et
`group` du fichier de configuration **sont** la configuration, lue au
démarrage.

## Ce que fait un rechargement dépend de ce qui a changé {#what-a-reload-does-depends-on-what-changed}

La ligne de partage passe par la **révocation** :

| ce qui a changé | ce qui se passe |
|---|---|
| **uniquement des ajouts** — une nouvelle personne dont l'arrivée ne modifie les listes développées d'aucun partage (un nouveau mot de passe d'application, une personne que les partages atteignent par des règles `oidc:`, ou pas du tout) | ajoutée **sur place**, serveur SMB en cours d'exécution compris, et **aucune connexion n'est touchée**. Un bridge qui crée des mots de passe d'application toute la journée ne dérange personne. |
| **tout retrait** — quelqu'un parti, un identifiant modifié, un partage dont les listes développées ont changé | une **nouvelle génération**, et les connexions de l'ancienne **fermées**, exactement comme pour une [modification d'administration]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}}). Une session qui a survécu au retrait de sa personne est précisément ce qu'un rechargement existe pour terminer. |
| **un annuaire illisible** — la base de données injoignable, une requête qui échoue | **rien ne change**. Une panne ne doit pas vider un serveur de fichiers, et « personne » est à quoi ressemble une lecture ratée. |

Une nouvelle personne qui rejoint un groupe que nomme un partage modifie *bel et bien* ce partage, et passe
par une nouvelle génération comme toute autre modification de celui-ci : SMB fixe les listes d'un partage
à son démarrage.

Un partage qui nomme quelqu'un de parti, ou un groupe qui n'existe plus, est servi
**sans eux** — le sens sûr — et c'est signalé, plutôt que refusé comme le refuserait un
démarrage : refuser conserverait les anciennes listes, et les anciennes listes sont
l'accès qui vient d'être retiré.

## Un groupe vide n'est pas tout le monde {#an-empty-group-is-not-everyone}

{{< callout type="error" >}}
**Un partage écrit pour `@engineers` dont le dernier ingénieur est parti n'est servi à personne**

Un `allow` vide signifie « quiconque s'authentifie ». Ce qui décide si
un partage est ouvert, c'est donc ce qui a été **écrit**, et non ce à quoi il se développe maintenant : un partage
écrit pour un groupe reste fermé quand ce groupe se vide. Un tel partage
n'est pas du tout proposé en SMB, dont l'`AllowUsers` vide se lirait comme
« tout le monde ».
{{< /callout >}}

## Le surveiller {#watching-it}

`fileshare_directory_reloads_total{result}` compte les rechargements qui n'ont rien trouvé
(`unchanged`), qui ont ajouté sur place (`added`), qui ont démarré une génération (`swapped`) ou
qui n'ont pas pu lire l'annuaire (`failed`) ; `fileshare_directory_people` indique combien
de personnes contenait la dernière lecture. Voir [santé et métriques]({{< relref "/administration/health.md" >}}).

{{< callout type="info" >}}
**Ce qu'un rechargement n'atteint pas**

Les personnes dont le fournisseur d'identité se porte garant par un jeton ou un certificat
ne sont pas dans l'annuaire : aucun rechargement ne les concerne. Les reprendre, **elles**,
est le rôle des [listes de révocation et des signaux partagés]({{< relref "/security/_index.md" >}}).
{{< /callout >}}
