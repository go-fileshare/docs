# Health and metrics

```hcl
metrics { listen = "127.0.0.1:9100" }   # or unix:///run/fileshare/metrics.sock
```

Three endpoints, served by
[go-net-health/endpoint](https://github.com/go-net-health/endpoint) on a
listener of their own — **never a port a protocol serves**, so the world that
reaches WebDAV does not reach them:

| | |
|---|---|
| `/healthz` | the process answers |
| `/readyz` | every protocol is bound **and** a generation is serving. `503`, and why, in the body, while starting, [applying a change](index.md#a-change-is-a-new-generation) or stopping |
| `/metrics` | Prometheus text format |

A unix socket is named by an absolute path, `unix:///like/this`; anything else
must be a `host:port`, or the block is refused.

## What is measured

| metric | what it counts |
|---|---|
| `fileshare_shares{origin}` | shares served, `origin` = `config` or `api` |
| `fileshare_shares_disabled` | shares taken offline through the admin API |
| `fileshare_generation` | how many lists of shares have been served since the start |
| `fileshare_connections_accepted_total{protocol}` | connections accepted |
| `fileshare_connections_open{protocol}` | connections open now |
| `fileshare_admin_changes_total{result}` | admin changes, `applied` or `refused` |
| `fileshare_admin_requests_total{method,code}` | admin API calls, by method and gRPC status |
| `fileshare_directory_people` | people the directory held at its last read |
| `fileshare_directory_reloads_total{result}` | [reloads](reload.md): `unchanged`, `added`, `swapped`, `failed` |
| `fileshare_revocation_list_age_seconds{list}` | age of a [revocation list](../security/revocation-lists.md)'s last good copy; `-1` when never fetched |
| `fileshare_revocation_list_fetch_failures_total{list}` | fetches of that list that failed |
| `fileshare_ssf_last_heard_seconds` | seconds since the [shared signals](../security/shared-signals.md) transmitter last answered; `-1` when never |
| `fileshare_ssf_revoked_subjects` | people whose earlier credentials the provider has revoked, as kept now |
| `fileshare_start_time_seconds` | when this process started |

plus the Go runtime and build information. The revocation and shared-signals
families appear only when the configuration has a list or an `ssf` block.

!!! danger "No metric names a share or a person"
    WebDAV answers **404** for a share somebody may not use, so that it is not
    confirmed to exist — and a scrape must not confirm it either. The labels are
    protocols, origins, outcomes and method names, never a share or a person.

## What to alert on

The revocation checks [fail closed](../security/index.md#every-one-of-them-fails-closed):
a list that is too old, or a transmitter not heard from, makes the server
**refuse** the credentials it governs. Alert before that happens, not after:

- `fileshare_revocation_list_age_seconds` approaching `ssh_krl_max_age` or
  `crl_max_age` (1h by default);
- `fileshare_ssf_last_heard_seconds` approaching the `ssf` block's `max_age`
  (10m by default);
- `fileshare_directory_reloads_total{result="failed"}` rising: the directory
  cannot be read, so [nothing changes](reload.md) — including removals.

!!! note "`--isolate`"
    A `metrics` block does not go with `--isolate` yet; the combination is
    refused.
