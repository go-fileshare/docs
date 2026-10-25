---
title: "管理 API"
linkTitle: "管理"
weight: 30
description: "サーバーの稼働中に共有の作成、変更、無効化、権限付与を行う gRPC 管理 API。待ち受ける場所と、拒否するもの。"
tags: [管理, 管理API, grpc]
---

v0.8.0 以降、サーバーの稼働中に共有を変更できます。`admin` ブロックを書くと gRPC サービスが有効になり、
再起動やファイルの編集なしに、共有の作成、変更、オフライン化ができます。

```hcl
admin {
  listen       = "unix:///run/fileshare/admin.sock"   # made 0600
  state_file   = "/var/lib/fileshare/shares.json"
  source_roots = ["/srv/images", "/data"]
}
```

ブロックを書かない限り、何も待ち受けません。

## できること {#what-it-does}

サービスは
[`fileshare.admin.v1.AdminService`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto)
です。

| | |
|---|---|
| `CreateShare`、`UpdateShare`、`DeleteShare` | イメージ、ディレクトリ、または(v0.21.0 以降)ボリュームから共有を定義する。`read_only` や `protocols` を変更する(ソースは変更できない。削除して別のものを作成する) |
| `DisableShare`、`EnableShare` | 共有をオフラインにする、およびオンラインに戻す |
| `Grant`、`Revoke` | サブジェクトに読み取りまたは書き込みのアクセス権を与える、または取り消す |
| `ListShares`、`GetShare` | すべての共有について、何を提供しているか、どのプロトコルで提供しているか、他のプロトコルで提供していない理由 |
| `ListUsers`、`ListGroups` | 権限付与で名指しできる対象。各ユーザーについては、その認証情報が応答できるプロトコル |
| `ReloadDirectory` | 人を今すぐ読み直す。[ディレクトリの再読み込み]({{< relref "/administration/reload.md" >}})を参照 |
| `GetServerInfo` | 名前、バージョン、起動時刻、世代、リスナー |
| `ListParents`、`CreateVolume`、`ResizeVolume`、`SnapshotVolume`、`DeleteVolume`、`GetVolume`、`ListVolumes` | 特権を持つプロビジョナー経由で作成されるストレージと、そこから提供される共有。[ボリューム]({{< relref "/administration/volumes.md" >}})を参照(v0.21.0 以降) |

権限付与で名指しできるのは、**ユーザー**、**グループ**(設定ファイルでは `@group`)、
**`oidc:groups:` の値**、**`oidc:user:` の名前**です。これらは設定ファイルで使うのと同じ語彙です。
[ID プロバイダーが名指しする人]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})を参照してください。

`grpc.health.v1` は同じリスナーで応答します。ブロックに `reflection = true` を指定すると、
`grpcurl` 用の gRPC サーバーリフレクションが有効になります。

### すべての変更は、提供にどう反映されたかを伝える {#every-change-says-what-serving-it-did}

変更は `Applied` メッセージで応答します。

```proto
message Applied {
  uint64 generation = 1;          // the generation now being served
  uint64 connections_closed = 2;  // how many the previous one had open, and closed
}
```

そのため、すべての RPC は共有そのものを返すのではなく、それぞれ専用の応答メッセージを持ちます。
`.proto` は、AIP-131 の「リソースを返す」よりも、[buf](https://buf.build) の `STANDARD` lint ルール
(fileshare の CI で `buf lint` により検査)に従っており、`Applied` を運ぶのがラッパーです。

## 待ち受ける場所と、呼び出せる人 {#where-it-listens-and-who-may-call-it}

{{< callout type="error" >}}
**TCP の場合は、相互 TLS か、何もなしかのどちらかです**

`tls_cert_file`、`tls_key_file`、`client_ca_file` の 3 つすべてが必要で、
**ループバックも例外ではありません**。ローカルユーザーなら誰でもループバックに到達でき、
この API は誰が誰のファイルを読めるかを決めるためです。Unix ソケットは **0600** で作成され、
そのパーミッションがアクセス制御になります。
{{< /callout >}}

```hcl
admin {
  listen         = "127.0.0.1:7443"
  tls_cert_file  = "/etc/fileshare/admin/server.pem"
  tls_key_file   = "/etc/fileshare/admin/server.key"
  client_ca_file = "/etc/fileshare/admin/clients.pem"
  state_file     = "/var/lib/fileshare/shares.json"
  source_roots   = ["/srv/images"]
}
```

リスナーは
[grpc-transports/control](https://github.com/grpc-transports/control) です。
すべての変更は、誰が行ったか(クライアント証明書の CN、またはソケットの相手の uid)とともに
記録されます。

## HTTPS 経由、OIDC トークン向け {#over-https-for-oidc-tokens}

v0.28.0 以降、`admin` ブロックに `web` ブロックがあれば、同じ API(同じ状態、同じ監査)が
HTTPS でも提供されます。**Connect、gRPC-Web、gRPC を 1 つのリスナーで**、
[`tls` ブロック]({{< relref "/security/tls.md" >}})の証明書を使って提供します。
Web UI とネイティブ UI が通信する先がこれです。

```hcl
admin {
  listen     = "unix:///run/fileshare/admin.sock"
  state_file = "/var/lib/fileshare/shares.json"
  web {
    listen = "0.0.0.0:8443"
    issuer "https://login.example.org" {
      audience = "fileshare-a"            # what THIS server is called there
      groups   = ["fileshare-admins"]
    }
    issuer "https://login.partner.example.org" {
      audience = "fileshare-a.partner"
      subjects = ["5b0c…"]                # a person is (issuer, sub)
    }
  }
}
```

| フィールド | |
|---|---|
| `listen` | TCP アドレス。`tls` ブロックがないと拒否される。ベアラートークンはパスワードと同じ価値を持つため |
| `issuer "<url>"` | ID プロバイダーごとに 1 ブロック(その `iss`)。同じものを 2 回指定すると拒否される |
| `audience` | トークンが名指しすべき `aud`。そのプロバイダーで**このサーバーが何と呼ばれているか**。必須 |
| `subjects`、`groups` | 呼び出せる人: `sub`、または groups クレーム内のグループ。少なくともどちらか一方が必要 |
| `jwks_url` | OIDC ディスカバリーを省略する。ディスカバリーを公開しないプロバイダー向け |
| `groups_claim` | グループを運ぶクレーム。既定は `groups` |

**誰に応答するか。** 呼び出しに応答するのは、ある発行者の検証器がそのベアラートークン
(署名、`iss`、このサーバーを名指しする `aud`、時刻)を受け入れ、**かつ**その発行者のブロックが
トークンのサブジェクトまたはそのグループのいずれかを名指ししている場合です。ある発行者に
列挙されたサブジェクトは、別の発行者では誰でもありません。人とは (発行者、`sub`) です。
検査はヘッダーだけで行われ、メッセージが 1 バイトもデコードされる前に実行されます。

**サーバーごとに 1 つのオーディエンス。** 複数のサーバー宛てのトークンは、そのうちのどれかが
他のサーバーに対して再利用できてしまいます(RFC 8707、RFC 9068)。そのため各サーバーが
独自のオーディエンスを持ち、複数のサーバーを操作する UI はサーバーごとに 1 つのトークンを要求します。

**拒否された呼び出し元が知ること。** 「拒否された」ということだけです。理由と、行われたすべての変更は、
`oidc=<issuer> <sub>` として監査出力に記録されます。

**CORS なし。** ブラウザーは直接呼び出しません。トークンを保持する UI 自身のサーバー経由で到達します。
リクエストは 1 MiB に制限され、接続のすべての段階にタイムアウトがあります。各発行者には鍵を読むために
**起動時に**接続します。到達できない発行者があると、その名前を示して起動が停止します。

### ブラウザーで: コンソール {#in-a-browser-the-console}

[go-fileshare/console](https://github.com/go-fileshare/console) は、この API を呼び出す管理
コンソールです。共有、詳細、権限付与、作成、有効化、無効化、削除、権限の付与と取り消し、
そして[ボリューム]({{< relref "/administration/volumes.md" >}})(作成、リサイズ、スナップショット、共有、
削除。データは決して破棄しない)を、すべてのサーバーにまたがって操作できます。
[go-fileshare/portal](https://github.com/go-fileshare/portal) の背後でブラウザー上で動作し、
portal が OpenID Connect で人をログインさせ、そのトークンを保持します。トークンはサーバーごとに 1 つで、
そのサーバーだけを宛先とします。ブラウザーがトークンを保持することはありません。

```hcl
# portal.hcl
server "paris" {
  url      = "https://fs-paris.example.org:8443"   # this web listener
  issuer   = "https://login.example.org"
  resource = "https://fs-paris.example.org/"       # this server's audience above
}
ui "console" { dir = "/usr/share/fileshare-console" }
```

同じコンソールは、portal なしで**ネイティブ**にも動作します。パブリッククライアントとして
([go-authn/rp](https://github.com/go-authn/rp)、RFC 8252)、システムのブラウザーとループバックの
リスナー経由でログインし、各サーバーを、そのサーバーだけを宛先とするトークンで呼び出します。
サーバーは自身の `console.hcl` に、ここと同じ `url` と `resource` で列挙します。

**ファイルエクスプローラー**の [go-fileshare/explorer](https://github.com/go-fileshare/explorer) は、
同じ portal の背後で動作します。人が利用できるすべての共有で、参照、アップロード、ダウンロード、
名前の変更、コピー、削除ができます。portal 経由(`/dav/<server>/...`)で各サーバーの WebDAV に、
そのサーバー宛てのトークンで到達し、返ってきたものを portal は**無害化して**提供します
(`Content-Security-Policy: sandbox`、添付ファイルとして)。共有上のページが portal として
実行されることは決してありません。fileshare **v0.29.0** 以降が必要です。その
[WebDAV]({{< relref "/protocols/webdav.md#the-root-and-uploads-in-pieces" >}})は、ルートで共有を一覧し、
分割アップロードを受け付けます。

```hcl
# portal.hcl、同じサーバー
server "paris" {
  # ... 上記と同じで、WebDAV も同様です:
  dav_url = "https://fs-paris.example.org"
}
ui "explorer" { dir = "/usr/share/fileshare-explorer" }
```

エクスプローラーも、コンソールと同様に portal なしで**ネイティブ**に動作します(explorer v0.2.0)。
パブリッククライアントとしてシステムのブラウザー経由でログインし、各サーバーの WebDAV を、
そのサーバーだけを宛先とするトークンで呼び出します。サーバーは `explorer.hcl` に、
それぞれの `dav_url`(https のみ)と `oidc` ブロックの `resource` とともに列挙します。
どちらのネイティブクライアントも、トークン付きのリダイレクトには従いません。Go の HTTP クライアントは、
同じホストへのリダイレクトであればスキームが異なっても `Authorization` を保持するため、
プレーンな `http` アドレスに `307` を返す誤設定のプロキシがあると、
トークンが平文で送られてしまうからです。

**失敗したアップロードは再開されます**(explorer v0.3.0)。パートファイルにはファイルの名前、
サイズ、更新時刻から付けた名前が使われるため、同じファイルをもう一度アップロードすると、
パートの末尾から続きが行われます。ただし、パートの最後の 64 KiB を読み戻して、
それがそのファイルのものであると確認できた場合に限ります。別のファイルの続きが行われることは
ありません。ネットワークやサーバーの障害ではパートが保持され、拒否(403、満杯の共有の 507)では
削除されます。

各サーバーの `audience` は、そのサーバーに対する portal の `resource` です。この API を使うために
独自のコンソールを書く必要はありません。このサーバー宛てのトークンを持つ Connect、gRPC-Web、gRPC の
クライアントであれば何でも使えます。

## 触れるものと触れないもの {#what-it-will-and-will-not-touch}

**自分が作成した共有を管理します。** 設定ファイルに書かれた共有は同じフィールドで一覧に表示されますが、
その定義の変更は `FAILED_PRECONDITION` で拒否されます。2 か所で定義された共有は誰も答えたくない
問いであり、その定義の場所はファイルだからです。API の共有は `state_file` にアトミックに書き込まれ、
次回の起動時に再び提供されます。`state_file` のないブロックが拒否されるのはそのためです。

**ソースは `source_roots` の下になければなりません。** 解決後の(リンクをたどり、`..` を取り除いた)
パスで判定され、保持されるのも解決後のパスです。`source_roots` がなければ、API は
**共有をまったく作成できません**。プロセスは `/dev` や `/etc` を読めますが、それらを呼び出し元に
渡すつもりは誰にもないからです。起動のたびに再検査され、`state_file` にある共有がルートの下に
なくなっていれば、その名前を示して起動が停止します。

**すべての共有には少なくとも 1 つの権限付与があります。** 権限付与のない共有は、認証した誰にでも
開放されます。ファイルでは意図的にそう書けますが、API 呼び出しで省略によってそうなるべきでは
ありません。そのため `CreateShare` には権限付与が必要で、最後の 1 つを取り消すことは拒否されます。
代わりに共有を削除してください。

**プロトコルが運べる名前。** 共有名は最大 80 文字で、空白で始まったり終わったりせず、
`.` や `..` ではなく、印字できない文字や `:*?"<>|{}%` のいずれも含みません。サブジェクトも
印字できない文字を含まず、1 つの共有が受け付ける権限付与は最大 1000 件です。それ以前に
グループの仕事でしょう。これらはいずれも v0.14.0 より前には受け付けられ、`state_file` に書き込まれた
うえで、起動のたびに致命的な問題になっていました。

**どの共有も、設定、状態ファイル、およびそれらが指すシークレットを含んではなりません。**
そこに書き込める人は、誰が何をできるかを書き換えられてしまいます。v0.17.0 以降は、
`authorized_keys_file`、`users` ブロックの `dsn_file` と `bind_password_file`、`ssf` ブロックの
`ca_file`、DSN が指す sqlite データベースも含みます。誰かの `authorized_keys` に自分の鍵を
追加できる書き込み者は、その人としてログインできてしまうからです。**また、共有が別の共有を
含んでもなりません**([ディレクトリ]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})を参照)。

**変更は、設定と同じように検査され、開かれ、書き込まれ、そのあとで提供されます。** サーバーが
受け入れられない変更(開けないイメージ、ディレクトリの誰も持たない名前)は拒否され、
提供中のものと書き込まれたものは元のままに保たれます。

## 共有の無効化 {#disabling-a-share}

**無効化は Samba の `available = no` に相当します。** 共有は定義されたままで、接続の試行はすべて失敗します。
開いている接続は閉じられ、イメージやディレクトリは**解放される**ので、オフラインの間にファイルを
置き換えられます。`EnableShare` で再び提供され、ソースを開けなくなっている場合は拒否されます。
`CreateShare` で、最初から無効化された共有を作成することもできます。

ほかのすべての変更と異なり、これは**設定ファイルの**共有にも適用されます。共有をオフラインにするのは
定義ではなく操作だからで、再起動をまたいで保持されます(状態ファイルに保存されます)。
[`fileshare check`]({{< relref "/configuration/check.md" >}})はオフラインのものを一覧します。

## 変更は新しい世代 {#a-change-is-a-new-generation}

{{< callout type="error" >}}
**変更はプロトコルサーバーを再起動し、その接続を閉じます**

SMB は接続を許可する人をツリー接続ごとに 1 回検査し、SFTP は人のツリーをログインごとに 1 回構築します。
そのため、失効後も生き残るセッションは、取り上げられたばかりのアクセスを保持し続けてしまいます。
「すでに接続していた人を除いて失効」は、失効ではありません。
{{< /callout >}}

したがって、変更は稼働中のサーバー*に対して*適用されるのではなく、新しいリストから構築された
新しいサーバー(**世代**)に置き換えられます。再起動と同じやり方ですが、ポートは手放しません。
ライブラリ側でほかの方法をとることはできませんでした。go-filesystems/smb には `Share` はあっても
`Unshare` がなく、nfs には `Export` はあっても `Unexport` がありません。

ポートはバインドされたままなので、変更中に接続してきたクライアントは拒否されず、**待たされます**。
共有が変わっていないイメージは、ドライバーを保持したまま、二度開かれることはありません。
クライアントは再接続します。これはすべての変更で支払う代価であり、API が
フィールドごとではなく呼び出しごとに 1 つの変更を適用するのもそのためです。

## 制限 {#limits}

- `--isolate` は、`admin` ブロックとも `metrics` ブロックともまだ併用できません。
  変更を適用できる単一のプロセスがないためで、この組み合わせは拒否されます。
- `-tags nogrpc` でビルドしたバイナリには管理 API がなく、`admin` ブロックのある設定は、
  管理 API なしで提供されるのではなく**拒否されます**。このタグで 11.7 MB を節約できます。
  [必要なものだけをビルドする]({{< relref "/operations/build-tags.md#the-admin-api-and-nogrpc" >}})を参照してください。

## `.proto` を変更する {#changing-the-proto}

`proto/` 以下の Go コードは生成されてコミットされているため、`go install` に `protoc` は不要です。
`.proto`(管理 API のもの、またはプロビジョナーの `proto/fileshare/provision/v1/provision.proto`)を
変更した後は、fileshare の CI が固定しているバージョン(protoc 34.1、protoc-gen-go v1.36.12、
protoc-gen-go-grpc v1.6.2)で両方を再生成してください。CI はそれらを再生成し、コミット済みのコードと
異なる場合や、生成コードがコミットされていない場合に失敗します。

```sh
protoc -I proto --go_out=. --go_opt=module=github.com/go-fileshare/fileshare \
  --go-grpc_out=. --go-grpc_opt=module=github.com/go-fileshare/fileshare \
  proto/fileshare/admin/v1/admin.proto proto/fileshare/provision/v1/provision.proto
```
