---
title: "ヘルスチェックとメトリクス"
weight: 20
description: "metrics ブロックの /healthz、/readyz、/metrics エンドポイントと、公開されるメトリクス、アラートの対象。"
tags: [管理, メトリクス]
---

```hcl
metrics { listen = "127.0.0.1:9100" }   # or unix:///run/fileshare/metrics.sock
```

3 つのエンドポイントを、[go-net-health/endpoint](https://github.com/go-net-health/endpoint) が
専用のリスナーで提供します。**プロトコルが使うポートでは決して提供しない**ため、
WebDAV に到達できる外部からはこれらに到達できません。

| | |
|---|---|
| `/healthz` | プロセスが応答する |
| `/readyz` | すべてのプロトコルがバインドされ、**かつ**世代が提供中である。起動中、[変更の適用中]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}})、停止中は `503` と、本文にその理由 |
| `/metrics` | Prometheus テキスト形式 |

Unix ソケットは `unix:///like/this` のように絶対パスで指定します。それ以外は `host:port` でなければならず、
そうでなければブロックは拒否されます。

## 計測されるもの {#what-is-measured}

| メトリクス | 数えるもの |
|---|---|
| `fileshare_shares{origin}` | 提供中の共有。`origin` は `config` または `api` |
| `fileshare_shares_disabled` | 管理 API でオフライン化された共有 |
| `fileshare_generation` | 起動以降に提供された共有リストの数 |
| `fileshare_connections_accepted_total{protocol}` | 受け付けた接続 |
| `fileshare_connections_open{protocol}` | 現在開いている接続 |
| `fileshare_admin_changes_total{result}` | 管理上の変更。`applied` または `refused` |
| `fileshare_admin_requests_total{method,code}` | 管理 API の呼び出し(メソッドと gRPC ステータスごと) |
| `fileshare_directory_people` | 直近の読み取り時にディレクトリが保持していた人数 |
| `fileshare_directory_reloads_total{result}` | [再読み込み]({{< relref "/administration/reload.md" >}}): `unchanged`、`added`、`swapped`、`failed` |
| `fileshare_revocation_list_age_seconds{list}` | [失効リスト]({{< relref "/security/revocation-lists.md" >}})の直近の正常なコピーの経過時間。一度も取得していない場合は `-1` |
| `fileshare_revocation_list_fetch_failures_total{list}` | そのリストの取得の失敗回数 |
| `fileshare_ssf_last_heard_seconds` | [共有シグナル]({{< relref "/security/shared-signals.md" >}})のトランスミッターが最後に応答してからの秒数。一度もない場合は `-1` |
| `fileshare_ssf_revoked_subjects` | プロバイダーが以前の認証情報を失効させた人のうち、現在保持しているもの |
| `fileshare_sftp_domain_grant_total{result}` | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}})が判定した SFTP 証明書: `granted`、`accepted_ungranted`、`refused_not_granted`、`refused_absent`、`refused_malformed` |
| `fileshare_start_time_seconds` | このプロセスの起動時刻 |

加えて、Go ランタイムとビルド情報も公開されます。失効と共有シグナルのファミリーは、設定にリストまたは
`ssf` ブロックがある場合のみ、ドメイン付与のファミリーは `ssh_domains` を設定した場合のみ、
管理リクエストのファミリーは `admin` ブロックがある場合のみ現れます。

{{< callout type="error" >}}
**共有名や人の名前を含むメトリクスはありません**

WebDAV は、誰かが利用できない共有に対して、その存在を確認させないよう **404** で応答します。
スクレイプも存在を確認させてはなりません。ラベルはプロトコル、origin、結果、メソッド名であり、
共有や人の名前が使われることはありません。
{{< /callout >}}

## アラートの対象 {#what-to-alert-on}

失効のチェックは[フェイルクローズ]({{< relref "/security/_index.md#every-one-of-them-fails-closed" >}})です。
古すぎるリストや、応答のないトランスミッターがあると、サーバーはそれが管理する認証情報を**拒否**します。
そうなった後ではなく、その前にアラートを出してください。

- `fileshare_revocation_list_age_seconds` が `ssh_krl_max_age` または
  `crl_max_age`(既定では 1h)に近づいている。
- `fileshare_ssf_last_heard_seconds` が `ssf` ブロックの `max_age`
  (既定では 10m)に近づいている。
- `fileshare_directory_reloads_total{result="failed"}` が増加している。ディレクトリを
  読み取れないため、削除を含めて[何も変更されません]({{< relref "/administration/reload.md" >}})。

{{< callout type="info" >}}
**`--isolate`**

`metrics` ブロックは、まだ `--isolate` とは併用できません。この組み合わせは拒否されます。
{{< /callout >}}
