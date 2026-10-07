---
title: "Health und Metriken"
weight: 20
description: "Die Endpunkte /healthz, /readyz und /metrics des metrics-Blocks, die Metriken, die sie bereitstellen, und worauf Sie alarmieren sollten."
tags: [administration, metriken]
---

```hcl
metrics { listen = "127.0.0.1:9100" }   # or unix:///run/fileshare/metrics.sock
```

Drei Endpunkte, bereitgestellt von
[go-net-health/endpoint](https://github.com/go-net-health/endpoint) auf einem
eigenen Listener – **nie ein Port, den ein Protokoll bedient**, sodass die Welt, die
WebDAV erreicht, sie nicht erreicht:

| | |
|---|---|
| `/healthz` | der Prozess antwortet |
| `/readyz` | jedes Protokoll ist gebunden **und** eine Generation wird bereitgestellt. `503`, mit dem Grund im Body, während des Starts, beim [Anwenden einer Änderung]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}}) oder beim Beenden |
| `/metrics` | Prometheus-Textformat |

Ein Unix-Socket wird durch einen absoluten Pfad benannt, `unix:///like/this`; alles andere
muss ein `host:port` sein, sonst wird der Block abgelehnt.

## Was gemessen wird {#what-is-measured}

| Metrik | was sie zählt |
|---|---|
| `fileshare_shares{origin}` | bereitgestellte Freigaben, `origin` = `config` oder `api` |
| `fileshare_shares_disabled` | über die Admin-API offline genommene Freigaben |
| `fileshare_generation` | wie viele Freigabelisten seit dem Start bereitgestellt wurden |
| `fileshare_connections_accepted_total{protocol}` | angenommene Verbindungen |
| `fileshare_connections_open{protocol}` | aktuell offene Verbindungen |
| `fileshare_admin_changes_total{result}` | Admin-Änderungen, `applied` oder `refused` |
| `fileshare_admin_requests_total{method,code}` | Aufrufe der Admin-API, nach Methode und gRPC-Status |
| `fileshare_directory_people` | Personen, die das Verzeichnis beim letzten Lesen enthielt |
| `fileshare_directory_reloads_total{result}` | [Neuladevorgänge]({{< relref "/administration/reload.md" >}}): `unchanged`, `added`, `swapped`, `failed` |
| `fileshare_revocation_list_age_seconds{list}` | Alter der letzten gültigen Kopie einer [Sperrliste]({{< relref "/security/revocation-lists.md" >}}); `-1`, wenn nie abgerufen |
| `fileshare_revocation_list_fetch_failures_total{list}` | fehlgeschlagene Abrufe dieser Liste |
| `fileshare_ssf_last_heard_seconds` | Sekunden, seit der Transmitter der [Shared Signals]({{< relref "/security/shared-signals.md" >}}) zuletzt geantwortet hat; `-1`, wenn nie |
| `fileshare_ssf_revoked_subjects` | Personen, deren frühere Zugangsdaten der Anbieter widerrufen hat, nach aktuellem Stand |
| `fileshare_sftp_domain_grant_total{result}` | SFTP-Zertifikate, zu denen [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}) befragt wurde: `granted`, `accepted_ungranted`, `refused_not_granted`, `refused_absent`, `refused_malformed` |
| `fileshare_start_time_seconds` | wann dieser Prozess gestartet ist |

dazu die Go-Laufzeit- und Build-Informationen. Die Familien für Widerruf und Shared Signals
erscheinen nur, wenn die Konfiguration eine Liste oder einen `ssf`-Block hat, die
Domain-Grant-Familie nur, wenn sie `ssh_domains` setzt, und die Familie der Admin-Anfragen
nur, wenn sie einen `admin`-Block hat.

{{< callout type="error" >}}
**Keine Metrik nennt eine Freigabe oder eine Person**

WebDAV antwortet mit **404** für eine Freigabe, die jemand nicht nutzen darf, damit ihre Existenz nicht
bestätigt wird – und ein Scrape darf sie ebenso wenig bestätigen. Die Labels sind
Protokolle, Herkünfte, Ergebnisse und Methodennamen, nie eine Freigabe oder eine Person.
{{< /callout >}}

## Worauf Sie alarmieren sollten {#what-to-alert-on}

Die Widerrufsprüfungen [sind ausfallsicher geschlossen (fail closed)]({{< relref "/security/_index.md#every-one-of-them-fails-closed" >}}):
Eine zu alte Liste oder ein Transmitter, von dem nichts zu hören ist, lässt den Server
die Zugangsdaten **ablehnen**, über die sie entscheiden. Alarmieren Sie, bevor das geschieht, nicht danach:

- `fileshare_revocation_list_age_seconds` nähert sich `ssh_krl_max_age` oder
  `crl_max_age` (standardmäßig 1h);
- `fileshare_ssf_last_heard_seconds` nähert sich `max_age` des `ssf`-Blocks
  (standardmäßig 10m);
- `fileshare_directory_reloads_total{result="failed"}` steigt: Das Verzeichnis
  kann nicht gelesen werden, also [ändert sich nichts]({{< relref "/administration/reload.md" >}}) – auch keine Entfernungen.

{{< callout type="info" >}}
**`--isolate`**

Ein `metrics`-Block passt noch nicht zu `--isolate`; die Kombination wird
abgelehnt.
{{< /callout >}}
