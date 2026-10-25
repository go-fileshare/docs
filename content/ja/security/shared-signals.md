---
title: "どのリストにも載らないものの失効: 共有シグナル"
linkTitle: "共有シグナル"
weight: 40
description: "共有シグナルのトランスミッターからの CAEP session-revoked イベントで、アクセストークン、OpenPubkey 証明書、その他のフェデレーション認証情報を失効させる。"
tags: [セキュリティ, 失効, 共有シグナル]
---

証明書には失効リストがあります。[KRL]({{< relref "/security/revocation-lists.md" >}})や
[CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})です。fileshare が単独で検証する
**アクセストークン**と、本人の鍵が署名した **OpenPubkey 証明書**にはありません。これらは、その人に
その後何があっても、期限が切れるまで有効です。

v0.13.0 以降、[OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html)
が、[CAEP](https://openid.net/specs/openid-caep-1_0-final.html) の **`session-revoked`** イベントによって
これを解決します。

```hcl
ssf {
  transmitter        = "https://bridge.example.org"          # its issuer
  audience           = "https://files.example.org"           # what this server is to it
  client_id          = "fileshare"                           # OAuth client credentials,
  client_secret_file = "/etc/fileshare/ssf.secret"           # scope "ssf" (or token_file)
  state_file         = "/var/lib/fileshare/revocations.json"
  max_age            = "10m"                                 # default; retain = "192h"
}
```

go-authn/bridge は、人や IdP が無効化されるとこれを送信します。fileshare はそれを**ポーリング**
(RFC 8936、SSF の既定)し、人ごとに、それが起きた**時刻**を保持します。

## それ以前に発行されたものはすべて、全プロトコルで拒否される {#everything-issued-before-is-refused-on-every-protocol}

その時点から、それ**以前に発行された**フェデレーション認証情報はすべて拒否され、**それらが開いた
セッションへの提供も停止されます**。これは KRL と同じゲートを通ります。「発行された」が何を意味するかは、
認証情報を運んだものによって異なります。

| プロトコル | 認証情報 | 発行時刻 |
|---|---|---|
| WebDAV | アクセストークン | その `iat` |
| SFTP | プロバイダーの SSH CA が署名した証明書 | 有効期間の開始(`ValidAfter`) |
| SFTP | OpenPubkey(opkssh)証明書 | ID トークンの `iat` |
| NFS | クライアント証明書([identity = "certificate"]({{< relref "/security/nfs-certificates.md" >}})) | その `NotBefore` |

プロバイダーがそれ**以降**に発行するものは有効です。再度有効化された人が締め出されることはありません。

{{< callout type="info" >}}
**同じ秒は「以前」として数えられます**

CAEP の `event_timestamp` とトークンの `iat` は秒単位の整数で、失効**時点またはそれ以前**に発行された
認証情報は拒否されます。そのため、失効と同じ秒に発行されたものも拒否され、無効化の直後に再度有効化された
人は、次の秒から有効な認証情報を得ます。これは意図的に保守的な側に倒したものです。発行時刻が
不明な場合は「ずっと前」ではなく、その人に対する失効が 1 つでもあれば拒否されます。
{{< /callout >}}

## イベントの対象となるのは誰か {#who-an-event-is-about}

サブジェクトは RFC 9493 の **`account`**(`acct:user@domain`、共有で使われる名前)、**`iss_sub`**、
**`email`**、またはそれらの **`aliases`** です。

**IdP 全体の無効化**は、CAEP の*テナント*サブジェクトとして、イベントに IdP のスコープを含めて届きます。
名前が `@` とそれらのいずれかである人(全体で比較され、`evil-univ-a.fr` は `univ-a.fr` ではない)で、
認証情報がそれ以前に発行された人は、**プロバイダーがもう記憶していない人を含めて**、すべて拒否されます。

## 受信側の認証方法 {#how-the-receiver-authenticates}

**OAuth クライアント認証情報**(RFC 6749 §4.4、スコープ `ssf`)を使います。`client_id` と
`client_secret_file` で、トランスミッター自身のトークンエンドポイント(その OpenID 設定で見つかるもの、
または `token_url`)からトークンを取得し、期限が切れる前に更新します。長期間有効なベアラートークンを
渡すトランスミッターのためには、代わりに `token_file` があります。どちらか一方で、両方指定も、
どちらも指定しないことも拒否されます。

| フィールド | 既定値 | |
|---|---|---|
| `transmitter` | | その発行者。**HTTPS のみ** |
| `audience` | | トランスミッターにとってこのサーバーが何であるか。必須。なければ、別の受信者宛ての SET がここで信じられてしまう |
| `client_id`、`client_secret_file` | | OAuth クライアント認証情報。両方セットで |
| `token_url` | 検出されたもの | HTTPS のみ。さもなければシークレットが平文でネットワークを横切る |
| `token_file` | | クライアント認証情報の代わりとなる長期トークン |
| `state_file` | | **必須**: 失効を書き留める場所 |
| `ca_file` | システムのもの | トランスミッターの認証局を固定する |
| `max_age` | `10m` | フェデレーション認証情報が拒否されるまでに、トランスミッターから応答がなくてよい期間。1 分以上 |
| `retain` | `192h` | 失効を保持する期間。169h 以上(v0.14.0 以降) |

失効は**確認応答の前に書き留められる**ため、確認応答された失効は再起動後も残ります。そして `retain` の間
保持されます。それが無効にしうるどの認証情報よりも長い期間です(192h は、プロバイダーが発行する最も寿命の長い
認証情報である 168h の SSH 証明書を超えています)。

失効させるフェデレーションされたものが何もない `ssf` ブロック、つまり `oidc` ブロックも NFS の
`identity = "certificate"` もない場合は拒否されます。

トランスポートは [github.com/hstern/go-ssf](https://github.com/hstern/go-ssf) です。ディスカバリー、
RFC 8936 のポーラー、SET の JWS 層を担います。イベントが**何を意味するか**は fileshare が決めます。

## フェイルクローズ {#it-fails-closed}

{{< callout type="error" >}}
**トランスミッターが沈黙している間、フェデレーション認証情報は拒否されます**

KRL と同様に、トランスミッターが `max_age` 以内にポーリングに応答していない間は、**すべての
フェデレーション認証情報**が拒否されます。「失効が届かなかった」と「届けられなかった」は同じに見えるからです。
アラートの対象は `fileshare_ssf_last_heard_seconds` で、`fileshare_ssf_revoked_subjects` は保持している
失効の数です。[ヘルスチェックとメトリクス]({{< relref "/administration/health.md#what-to-alert-on" >}})を
参照してください。
{{< /callout >}}

## エンドツーエンドで検証済み {#checked-end-to-end}

go-authn/bridge の相互運用レーン
([#12](https://github.com/go-authn/bridge/pull/12)、
[#17](https://github.com/go-authn/bridge/pull/17))は、fileshare v0.13.0 を本物の bridge に対して
判定します。KRL、NFS 上の CRL、SSF で失効した WebDAV トークン、スコープで無効化された IdP、
そして SSF 上の opkssh、つまりどの失効リストも届かないケースです。
