---
title: "Santé et métriques"
weight: 20
description: "Les points de terminaison /healthz, /readyz et /metrics du bloc metrics, les métriques qu'ils exposent, et sur quoi alerter."
tags: [administration, métriques]
---

```hcl
metrics { listen = "127.0.0.1:9100" }   # or unix:///run/fileshare/metrics.sock
```

Trois points de terminaison, servis par
[go-net-health/endpoint](https://github.com/go-net-health/endpoint) sur un
écouteur qui leur est propre — **jamais un port que sert un protocole** : le monde qui
atteint WebDAV ne les atteint donc pas :

| | |
|---|---|
| `/healthz` | le processus répond |
| `/readyz` | chaque protocole est lié **et** une génération est servie. `503`, avec la raison dans le corps, pendant le démarrage, l'[application d'une modification]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}}) ou l'arrêt |
| `/metrics` | format texte Prometheus |

Un socket unix se nomme par un chemin absolu, `unix:///like/this` ; tout le reste
doit être un `host:port`, sinon le bloc est refusé.

## Ce qui est mesuré {#what-is-measured}

| métrique | ce qu'elle compte |
|---|---|
| `fileshare_shares{origin}` | partages servis, `origin` = `config` ou `api` |
| `fileshare_shares_disabled` | partages mis hors ligne par l'API d'administration |
| `fileshare_generation` | combien de listes de partages ont été servies depuis le démarrage |
| `fileshare_connections_accepted_total{protocol}` | connexions acceptées |
| `fileshare_connections_open{protocol}` | connexions ouvertes en ce moment |
| `fileshare_admin_changes_total{result}` | modifications d'administration, `applied` ou `refused` |
| `fileshare_admin_requests_total{method,code}` | appels à l'API d'administration, par méthode et par statut gRPC |
| `fileshare_directory_people` | personnes que contenait l'annuaire à sa dernière lecture |
| `fileshare_directory_reloads_total{result}` | [rechargements]({{< relref "/administration/reload.md" >}}) : `unchanged`, `added`, `swapped`, `failed` |
| `fileshare_revocation_list_age_seconds{list}` | âge de la dernière bonne copie d'une [liste de révocation]({{< relref "/security/revocation-lists.md" >}}) ; `-1` si elle n'a jamais été récupérée |
| `fileshare_revocation_list_fetch_failures_total{list}` | récupérations de cette liste qui ont échoué |
| `fileshare_ssf_last_heard_seconds` | secondes depuis la dernière réponse de l'émetteur de [signaux partagés]({{< relref "/security/shared-signals.md" >}}) ; `-1` s'il n'a jamais répondu |
| `fileshare_ssf_revoked_subjects` | personnes dont le fournisseur a révoqué les identifiants antérieurs, telles que conservées actuellement |
| `fileshare_sftp_domain_grant_total{result}` | certificats SFTP soumis à [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}) : `granted`, `accepted_ungranted`, `refused_not_granted`, `refused_absent`, `refused_malformed` |
| `fileshare_start_time_seconds` | quand ce processus a démarré |

plus les informations du runtime Go et de compilation. Les familles de révocation et de signaux partagés
n'apparaissent que lorsque la configuration a une liste ou un bloc `ssf`, la
famille des délégations de domaine seulement quand elle définit `ssh_domains`, et la famille des requêtes
d'administration seulement quand elle a un bloc `admin`.

{{< callout type="error" >}}
**Aucune métrique ne nomme un partage ou une personne**

WebDAV répond **404** pour un partage que quelqu'un ne peut pas utiliser, afin que son existence ne soit pas
confirmée — et une collecte ne doit pas la confirmer non plus. Les étiquettes sont des
protocoles, des origines, des résultats et des noms de méthodes, jamais un partage ou une personne.
{{< /callout >}}

## Sur quoi alerter {#what-to-alert-on}

Les vérifications de révocation [refusent en cas de doute]({{< relref "/security/_index.md#every-one-of-them-fails-closed" >}}) :
une liste trop ancienne, ou un émetteur dont on n'a plus de nouvelles, fait que le serveur
**refuse** les identifiants qu'il gouverne. Alertez avant que cela n'arrive, pas après :

- `fileshare_revocation_list_age_seconds` qui approche `ssh_krl_max_age` ou
  `crl_max_age` (1h par défaut) ;
- `fileshare_ssf_last_heard_seconds` qui approche le `max_age` du bloc `ssf`
  (10m par défaut) ;
- `fileshare_directory_reloads_total{result="failed"}` qui augmente : l'annuaire
  ne peut pas être lu, donc [rien ne change]({{< relref "/administration/reload.md" >}}) — suppressions comprises.

{{< callout type="info" >}}
**`--isolate`**

Un bloc `metrics` ne va pas encore avec `--isolate` ; la combinaison est
refusée.
{{< /callout >}}
