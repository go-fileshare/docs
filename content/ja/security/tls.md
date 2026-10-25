---
title: "TLS と ACME による証明書"
weight: 10
description: "ファイルまたは ACME CA の証明書を使った WebDAV、S3、NFS の TLS 提供と、パスワードを使う WebDAV の平文提供が拒否される理由。"
tags: [セキュリティ, tls, acme]
---

v0.9.0 以降、**WebDAV と S3 は HTTPS で、NFS は RPC-with-TLS**(RFC 9289)で提供できます。
`serve` ブロックに `tls = true` を指定し、証明書の取得元を示す `tls` ブロックを 1 つ書きます。

## ファイルから {#from-files}

```hcl
tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true
}
```

ファイルは**変更されると再読み込み**されるため、他のツールが書き込んだ更新分は再起動なしで反映されます。
証明書は [go-authn/servercert](https://github.com/go-authn/servercert) のものです。

## ACME CA から {#from-an-acme-ca}

既定では Let's Encrypt です。HARICA 経由の GÉANT TCS のように、**外部アカウントバインディング**で
利用者を把握する CA も使えます。

```hcl
tls {
  acme {
    directory_url     = "https://acme-v02.harica.gr/acme/<uuid>/directory"   # the Server URL cm.harica.gr shows
    domains           = ["files.example.org"]
    cache_dir         = "/var/lib/fileshare/acme"
    eab_key_id        = "…"
    eab_hmac_key_file = "/etc/fileshare/tls/eab.key"   # a secret: a file, never the config
  }
}
```

| フィールド | |
|---|---|
| `directory_url` | CA の ACME ディレクトリ。空なら Let's Encrypt |
| `domains` | 要求する名前(必須) |
| `cache_dir` | アカウントと証明書を保管する場所(必須) |
| `email` | アカウントの連絡先(省略可) |
| `eab_key_id`、`eab_hmac_key_file` | 外部アカウントバインディング。HMAC 鍵はシークレットなのでファイルから読み込む。CA が渡す形式の base64url(標準の base64 も読み取れる)で、それ以外は拒否される |
| `http_challenge` | http-01 に応答する場所(例: `"0.0.0.0:80"`) |

### ACME が使えるかどうかは、CA が名前をどう検証するかで決まる {#whether-acme-can-work-is-decided-by-how-the-ca-checks-the-name}

| チャレンジ | CA の接続先 | したがって |
|---|---|---|
| tls-alpn-01(RFC 8737) | ポート **443** | TLS プロトコルを 443 で提供する必要がある |
| http-01(RFC 8555 §8.3) | ポート **80** | `http_challenge = "0.0.0.0:80"` で応答する。またはポート 80 の転送先のアドレス |
| なし | なし | ドメインが**事前検証済み**の CA アカウントはチャレンジを要求しない。GÉANT TCS(OV 証明書)向けの HARICA Enterprise **Admin** アカウントがこれにあたり、**外部から誰も到達できない**サーバーでも証明書を取得できる |

最後の行は、キャンパスネットワーク内のファイルサーバーにとって重要です。機関がすでに HARICA で
ドメインを検証済みの GÉANT TCS アカウントがあれば、インターネットに向けてポートを開く必要は一切ありません。

#### HARICA(GÉANT TCS): どのアカウントか、そして CAA {#harica-gant-tcs-which-account-and-caa}

HARICA の ACME アカウントには 2 種類あり、チャレンジを省略できるのはそのうちの 1 つだけです。

- **Enterprise Admin** アカウントは、機関が検証済みのドメインに対して **OV** 証明書を発行します。
  **チャレンジなし**で、外部から誰も到達できないサーバーにはこれを使います。
- **Enterprise User** アカウントは **DV** 証明書を発行します。DV とは、CA が注文のたびに名前を検査することを意味し、
  上記のとおり tls-alpn-01 または http-01 を使います。

どちらの場合も、ドメインの **CAA** レコードがあれば、`harica.gr` を許可していなければなりません。
そうでなければ注文は拒否されます。

#### 証明書が要求されるタイミング {#when-the-certificate-is-asked-for}

**起動時**(v0.27.0 以降): リスナーが立ち上がると、設定されたすべてのドメインの証明書が取得され、
またはキャッシュディレクトリから読み戻されます。失敗してもサーバーは停止しません。提供を続けながら
1 分後に再試行し、最大 1 時間まで倍々にします。そのホストを名指しする最初の TLS 接続も、いずれにしても
再度要求します。証明書は期限の 30 日前、または有効期間がそれより短い場合はその最後の 3 分の 1 の時点で
更新されます。

**SNI を送らないクライアント**(IP アドレスで接続するもの)には、**最初の**ドメインの証明書が渡されます
(v0.27.0 以降)。以前はハンドシェイクが失敗していました。クライアントは依然として、その証明書を
ダイヤルしたアドレスに対して検査するため、その名前を信頼するよう指示されたクライアントを助けるだけで、
誰も欺きません。

{{< callout type="info" >}}
**ALPN**

ACME を使うと、サーバーの ALPN リストには `acme-tls/1` が入り、Go のサーバーは自身のリストと共通点のない
リストを持つクライアントを拒否します。そのため WebDAV と S3 には `http/1.1` が**追加**され
(すべてのブラウザーと WebDAV クライアントが提示します)、NFS には RFC 9289 §5.2 が RPC-with-TLS を
識別するものとして定める `sunrpc` が追加されます。`h2` は追加しません。これらのリスナーでは
HTTP/2 を提供しておらず、提示するのは偽りになるためです。`acme-tls/1` はすべてのリスナーに残るので、
CA がどこに到達しても tls-alpn-01 に応答できます。
{{< /callout >}}

## WebDAV は平文では提供されない {#webdav-is-not-served-in-the-clear}

{{< callout type="error" >}}
**v0.9.0 での破壊的変更: 到達可能なアドレスでパスワードを使う WebDAV は拒否されます**

HTTP Basic では、すべてのリクエストが base64 化されたパスワードそのものであり、ベアラートークンも同等の
価値を持ちます。そのため、他のマシンから到達できるアドレスで、誰かが認証する状況において、**TLS なし**で
WebDAV を提供する設定は、**警告ではなく拒否**されます。警告は流れて消えますが、漏れたパスワードは
戻ってきません。

```
webdav on 0.0.0.0:8080 would carry passwords in the clear: serve it with
tls = true, or -- when TLS is terminated in front of it, by a proxy -- say
plaintext = true
```
{{< /callout >}}

v0.9.0 より前に `serve "webdav" { addr = "0.0.0.0:8080" }` で起動していた設定は、それ以降は起動しません。
対処法は 2 つあります。

```hcl
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true            # with a tls block
}
```

または、リバースプロキシが前段で TLS を終端し、平文で転送する場合は次のようにします。

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true       # TLS is terminated in front of this, on purpose
}
```

**ループバック**アドレス(`127.0.0.1`、`::1`、`localhost`)と、**認証する相手のいない**サーバー
(`user`、`users`、`oidc` ブロックのいずれもない)は影響を受けません。`addr` のない `serve` ブロックは
ループバックに置かれるため、これも影響を受けません。

`plaintext` はパスワードを送るプロトコルである WebDAV 専用で、`plaintext` と `tls` を同時に指定することは
拒否されます。どちらか一方です。

## `check` が伝えること {#what-check-says}

[`fileshare check`]({{< relref "/configuration/check.md" >}}) は、プロトコルごとに、何が暗号化されていて、
何が**意図的に**平文であるかを出力します。

```
webdav: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem
nfs: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem; clients present one signed by /etc/fileshare/tls/clients.pem (the machine, not the person)
```

また、プロキシの背後の WebDAV では次のようになります。

```
webdav: in the clear on 10.0.0.5:8080, on purpose (plaintext = true): TLS must be terminated in front of it
```

## プロトコルごと {#per-protocol}

| プロトコル | `tls = true` | |
|---|---|---|
| WebDAV | yes | HTTPS |
| S3 | yes | HTTPS |
| NFS | yes | RFC 9289。後述 |
| SMB | **拒否** | SMB 3 は独自の鍵で暗号化する |
| SFTP | **拒否** | SFTP は SSH である |

## TLS 上の NFS が証明するのはマシンであり、人ではない {#nfs-over-tls-proves-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

`client_ca_file` は、その認証局の証明書をクライアントに提示させます。これは**どのホストがマウントしているか**を
示します。RFC 9289 はユーザー認証を以前のままにしているため、内側の uid は依然として `AUTH_SYS` が主張する
ものです。したがって、利用できる人を指定した共有は、`kerberos` ブロックがない限り、または同じ証明書の
別の使い方である[証明書による ID]({{< relref "/security/nfs-certificates.md" >}})がない限り、
NFS では**依然として拒否されます**。

TLS は**提供されるだけで必須ではありません**。TLS を要求しないクライアントには、引き続き開放された共有が
提供されます。`tls = true` のない `client_ca_file` は、何も検査しないため拒否され、NFS 以外のプロトコルの
`client_ca_file` も拒否されます。

## ほかに拒否されるもの {#what-else-is-refused}

- `tls` ブロックのない `tls = true`: 証明書の取得元を示すものがありません。
- どの `serve` ブロックからも使われない `tls` ブロック: 使うものがありません。
- 待ち受けるアドレスではない `http_challenge`。
