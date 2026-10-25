---
title: "WebDAV — Basic、またはベアラートークン"
linkTitle: "WebDAV: Basic、またはベアラートークン"
weight: 20
description: "HTTP Basic または OIDC ベアラートークンを使う WebDAV。必須となる TLS と、トークンがここで人になる仕組み。"
tags: [プロトコル, webdav, oidc]
---

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

HTTP Basic またはベアラートークンを、ファイルまたは ACME による[TLS]({{< relref "/security/tls.md" >}})上で使います。

{{< callout type="error" >}}
**v0.9.0 以降、平文では提供されません**

HTTP Basic ではすべてのリクエストがパスワードそのものであり、ベアラートークンもそれと同等の価値を持ちます。
他のマシンから到達できるアドレスで、誰かが認証する状況において、**TLS なし**の WebDAV は
**起動時に拒否されます**。このページがかつて示していた例
`serve "webdav" { addr = "0.0.0.0:8080" }` は、もう起動しません。前段のプロキシが TLS を終端する場合は、
そのことを明示してください。

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true
}
```

ループバックと、認証する相手のいないサーバーは影響を受けません。
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})を参照してください。
{{< /callout >}}

{{< callout type="info" >}}
**403 ではなく 404**

人が利用できない共有は **404** で応答します。存在を確認させません。
{{< /callout >}}

## トークンを運べる唯一のプロトコルでの、トークン {#a-token-over-the-one-protocol-that-can-carry-one}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
}
```

ブラウザーはトークンを持っていますが、パスワードを持っていません。そのため WebDAV は
`Authorization: Bearer` を受け付けます。ローカルの人もいる場合、送信するチャレンジは**両方**を提示し、
クライアントは応答できるほうを選びます。トークンは
[go-authn/oidc](https://github.com/go-authn/oidc) が検証します。署名、発行者、オーディエンス、有効期限です。

{{< callout type="error" >}}
**ベアラートークンは WebDAV だけ**

SMB は NTLMv2、SFTP は鍵または証明書、NFS は何もなし、S3 は SigV4 署名で認証し、どれにも
`Authorization` ヘッダーを載せる場所がありません。これはプロトコルについての事実であり、
このプログラムの制約ではありません。

プロバイダーの言葉は、代わりに証明書で **SFTP** に届きます。[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})を参照してください。
プロバイダーを指定しているのに、WebDAV もそのような SFTP も提供しない設定は、
**起動せずに拒否されます**。
{{< /callout >}}

{{< callout type="error" >}}
**トークンが示すのは、プロバイダーが誰だと考えているかです。このサーバーにその人の共有があるとは限りません。**

ここのどのソースも知らない名前の有効なトークンは拒否されます。「あなたを知らない」を安全に読むと、
「あなたは許可されている」にはなりません。ローカルアカウントにも同じ名前がある場合でも、
`local_names = true` でプロバイダーの名前がこのサーバーの名前と同じだと明示していない限り、
そのアカウントにはなりません(v0.20.0 以降は既定でオフ)。それがなければ、`oidc:` ルールがその人を
名指ししていない限り、そのようなトークンも拒否されます。
{{< /callout >}}

プロバイダーが**そのままディレクトリである**サイトは、そう明示します。

```hcl
oidc {
  issuer    = "https://login.example.org"
  audience  = "fileshare"
  trust_all = true      # everybody that provider vouches for, not just these people
}
```

ここには**ログインフローはありません**。リダイレクトも、クライアントシークレットも、クッキーもありません。
これはリソースサーバーです。

トークンが与える**名前**は `preferred_username`、または `username_claim` が指すクレームです。
`preferred_username` がない場合は、プロバイダーが `email_verified: true` と言っているときに限ってメール、
そうでなければ `sub` になります。`username_claim = "email"` の場合、メールが検証されていないトークンは
拒否されます。これは WebDAV でも、SFTP での opkssh ログインでも同じです。検証されるまで、
メールは本人が入力したものにすぎず、それを載せた署名付きトークンは、誰も確認していないアドレスとして
結びついてしまいます(OpenID Connect Core 5.1、go-authn/oidc v0.2.0、fileshare v0.16.3)。

プロバイダーの人のうち誰が共有を得るか(グループ、名指しした人、機関全体)は、`oidc:` ルールと
`domains` で記述します。[ID プロバイダーが名指しする人]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})を参照してください。

## トークンは、何かが変えない限り、その人より長く生きる {#a-token-outlives-the-person-unless-something-says-otherwise}

トークンはここで単独で検証され、期限が切れるまで、その人にその後何があっても有効です。
どの失効リストもそれを名指しできません。[`ssf` ブロック]({{< relref "/security/shared-signals.md" >}})を使うと、
プロバイダーからの CAEP `session-revoked` によって、それ以前に発行された(`iat`)すべてのトークンが無効になります。

## ルートと、分割アップロード {#the-root-and-uploads-in-pieces}

**ルートは共有を一覧します**(v0.29.0 以降)。`/` にいるブラウザーは、開ける共有のページを受け取ります。
`/` への `PROPFIND` は、共有をコレクションとして受け取る(`207`)ため、ファイルマネージャーは
1 つの共有ではなくサーバー全体をマウントできます。Depth `infinity` は拒否され
(RFC 4918 が許すとおり `403`、`propfind-finite-depth`)、ルートに書き込むものも拒否されます(`405`)。
v0.28 までは、`/` へのすべてのメソッドがそのページを受け取っていました。

**大きなアップロードは分割できます**(v0.29.0 以降)。1 つの `PUT` は最大 1 GiB まで丸ごとメモリに保持されるため、
ストリーミングできないクライアント(ブラウザー、[ファイルエクスプローラー]({{< relref "/administration/_index.md" >}}))は、
空の `PUT` を送り、続いて前の分が終わった位置からちょうど始まる `Content-Range` 付きで各部分を送り、
最後に本来の名前への `MOVE` を行います。終端を越えて始まる範囲は、依然として `416` です。
誰も送っていないバイトが残ってしまうためです。
