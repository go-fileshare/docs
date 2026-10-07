---
title: "Salud y métricas"
weight: 20
description: "Los endpoints /healthz, /readyz y /metrics del bloque metrics, las métricas que exponen y sobre qué alertar."
tags: [administración, métricas]
---

```hcl
metrics { listen = "127.0.0.1:9100" }   # or unix:///run/fileshare/metrics.sock
```

Tres endpoints, servidos por
[go-net-health/endpoint](https://github.com/go-net-health/endpoint) en una escucha
propia —**nunca un puerto que sirve un protocolo**—, de modo que quien alcanza WebDAV
no los alcanza:

| | |
|---|---|
| `/healthz` | el proceso responde |
| `/readyz` | todos los protocolos están vinculados **y** hay una generación sirviendo. `503`, con el motivo en el cuerpo, durante el arranque, al [aplicar un cambio]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}}) o al detenerse |
| `/metrics` | formato de texto de Prometheus |

Un socket unix se nombra con una ruta absoluta, `unix:///like/this`; cualquier otra
cosa debe ser un `host:port`, o el bloque se rechaza.

## Lo que se mide {#what-is-measured}

| métrica | qué cuenta |
|---|---|
| `fileshare_shares{origin}` | recursos compartidos servidos, `origin` = `config` o `api` |
| `fileshare_shares_disabled` | recursos compartidos retirados de servicio mediante la API de administración |
| `fileshare_generation` | cuántas listas de recursos compartidos se han servido desde el arranque |
| `fileshare_connections_accepted_total{protocol}` | conexiones aceptadas |
| `fileshare_connections_open{protocol}` | conexiones abiertas ahora |
| `fileshare_admin_changes_total{result}` | cambios de administración, `applied` o `refused` |
| `fileshare_admin_requests_total{method,code}` | llamadas a la API de administración, por método y estado gRPC |
| `fileshare_directory_people` | personas que contenía el directorio en su última lectura |
| `fileshare_directory_reloads_total{result}` | [recargas]({{< relref "/administration/reload.md" >}}): `unchanged`, `added`, `swapped`, `failed` |
| `fileshare_revocation_list_age_seconds{list}` | antigüedad de la última copia válida de una [lista de revocación]({{< relref "/security/revocation-lists.md" >}}); `-1` si nunca se descargó |
| `fileshare_revocation_list_fetch_failures_total{list}` | descargas fallidas de esa lista |
| `fileshare_ssf_last_heard_seconds` | segundos desde la última respuesta del transmisor de [señales compartidas]({{< relref "/security/shared-signals.md" >}}); `-1` si nunca |
| `fileshare_ssf_revoked_subjects` | personas cuyas credenciales anteriores ha revocado el proveedor, tal como se conservan ahora |
| `fileshare_sftp_domain_grant_total{result}` | certificados SFTP sobre los que se consultó [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}): `granted`, `accepted_ungranted`, `refused_not_granted`, `refused_absent`, `refused_malformed` |
| `fileshare_start_time_seconds` | cuándo arrancó este proceso |

más la información del runtime de Go y de compilación. Las familias de revocación y
de señales compartidas solo aparecen cuando la configuración tiene una lista o un
bloque `ssf`, la de concesión de dominio solo cuando establece `ssh_domains`, y la de
peticiones de administración solo cuando tiene un bloque `admin`.

{{< callout type="error" >}}
**Ninguna métrica nombra un recurso compartido ni a una persona**

WebDAV responde **404** para un recurso compartido que alguien no puede
usar, para no confirmar que existe, y una recogida de métricas tampoco debe
confirmarlo. Las etiquetas son protocolos, orígenes, resultados y nombres de
método, nunca un recurso compartido ni una persona.
{{< /callout >}}

## Sobre qué alertar {#what-to-alert-on}

Las comprobaciones de revocación [fallan en modo cerrado]({{< relref "/security/_index.md#every-one-of-them-fails-closed" >}}):
una lista demasiado antigua, o un transmisor del que no se sabe nada, hace que el
servidor **rechace** las credenciales que gobierna. Alerte antes de que ocurra, no
después:

- `fileshare_revocation_list_age_seconds` acercándose a `ssh_krl_max_age` o a
  `crl_max_age` (1h por defecto);
- `fileshare_ssf_last_heard_seconds` acercándose al `max_age` del bloque `ssf`
  (10m por defecto);
- `fileshare_directory_reloads_total{result="failed"}` en aumento: el directorio no
  se puede leer, así que [nada cambia]({{< relref "/administration/reload.md" >}}), tampoco las bajas.

{{< callout type="info" >}}
**`--isolate`**

Un bloque `metrics` todavía no es compatible con `--isolate`; la
combinación se rechaza.
{{< /callout >}}
