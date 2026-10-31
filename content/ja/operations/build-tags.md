---
title: "必要なものだけをビルドする"
weight: 10
description: "プロトコル、ユーザーディレクトリ、管理 API、プロビジョナーをバイナリから除外するビルドタグと、それぞれで節約できるサイズ。"
tags: [運用, ビルド]
---

各プロトコルはビルドタグの背後にあり、タグを指定するとそのプロトコルは**完全に**除外されます。
リスナーも、パーサーも、依存関係も、コードもありません。

```sh
go install -tags nonfs,nowebdav,nosftp,nos3 github.com/go-fileshare/fileshare@latest   # SMB only
go build   -tags nosmb,nonfs,nosftp,nos3 .                                            # WebDAV only
```

**人**の取得元も、それ専用のタグの背後にあります。`nosql` は 3 つのデータベースドライバーを、
`noldap` は LDAP クライアントを除外します。

| ビルド | サイズ |
|---|---|
| すべて | 31.2 MB |
| `-tags noldap` | 30.9 MB |
| `-tags nosftp` | 30.6 MB |
| `-tags noopenpubkey`(SFTP での opkssh ログインなし) | 約 2 MB 減 |
| `-tags nos3` | 31.1 MB |
| `-tags nonfs,nowebdav,nosftp,nos3`(SMB のみ) | 28.1 MB |
| `-tags nopartitioned`(apfs、btrfs、xfs、zfs なし) | 29.2 MB |
| `-tags nosql` | 20.2 MB |
| `-tags nosql,noldap` | 19.8 MB |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp,nos3` | 16.2 MB |

これらは fileshare の README に記載されている数値で、管理 API が加わる前に計測されたものです。
下の表がそれ以降の計測です。

## 管理 API と `nogrpc` {#the-admin-api-and-nogrpc}

[管理 API]({{< relref "/administration/_index.md" >}})は gRPC と protobuf を持ち込み、それらは
`nogrpc` の背後にあります(linux/amd64、2026-09-29 に計測。このとき上の行は、すべて含めると
34.1 MB に増えていました)。

| ビルド | サイズ |
|---|---|
| すべて、管理 API 付き | 46.1 MB |
| `-tags nogrpc` | 34.4 MB |
| `-tags nosql,noldap,nogrpc` | 22.5 MB |

gRPC のコストは **11.7 MB** です。これは下の[プラグインの計測](#why-not-subprocess-plugins)を、
今回は独自の理由があって取り直したものです。共有をファイルで管理するサイトは、
これを抱える必要がありません。

{{< callout type="warning" >}}
**`nogrpc` ビルドで `admin` ブロックを指定すると拒否されます**

管理 API なしで提供するのではなく、拒否します。API を求めておきながらそれなしで起動する設定は、
書いてあることと異なる動作をするためです。
{{< /callout >}}

`nogrpc` は管理 API を除外し、それに伴って[プロビジョナー]({{< relref "/administration/volumes.md" >}})も除外します。
[ヘルスチェックとメトリクス]({{< relref "/administration/health.md" >}})は通常の HTTP なので、残ります。

`-tags noprovisioner` はプロビジョナー、つまり特権を持つ `fileshare provisioner` ロールだけを除外し、
go-fsctl の zfs、btrfs、projquota も一緒に除外します。ボリュームを一切提供しないサーバーは、
それを作成するコードを抱える必要がありません。そのようなビルドの `fileshare provisioner` は、
「不明なコマンド」ではなく、その旨を伝えます。

`nosql` は群を抜いて大きな節約手段です。PostgreSQL、MySQL、SQLite を合わせると **11.7 MB** になり、
このプログラムのすべてのプロトコルを合わせたものより大きくなります。ユーザーをファイルで管理する
サイトには、これが向いています。ドライバーはそれを使うライブラリではなくコマンドによって
インポートされるため、この選択はフォークではなくビルドタグで行えます。

{{< callout type="info" >}}
**このバイナリがビルドされていないプロトコル**

そのプロトコルを指定した設定には、「そのようなプロトコルはありません」ではなく、*そうである*ことが
伝えられます。これは、タイプミスとビルドタグの違いです。`noopenpubkey` の下の `opkssh_client_id` と、
`nogrpc` の下の `admin` ブロックでも同様です。
{{< /callout >}}

## WebAssembly コンテナとして (`wasip1`) {#as-a-webassembly-container-wasip1}

v0.32.0 から、fileshare は `GOOS=wasip1 GOARCH=wasm` 向けにビルドでき、WebDAV を
WASI コンテナとして提供します。たとえば
[go-wasmcontainers](https://github.com/go-wasmcontainers) エンジン (`wctr`) で、
ネイティブまたはブラウザのタブ内で動かせます。

```sh
GOOS=wasip1 GOARCH=wasm go build -trimpath \
  -tags nosql,noopenpubkey,nosmb,nonfs,nos3,nosftp,nogrpc,nopartitioned,noprovisioner \
  -o fileshare.wasm .                              # WebDAV のみ: gzip で 6.5 MB
```

- `nosql` と `noopenpubkey` は必須です。SQLite ドライバと OpenPubkey の依存関係は
  wasip1 ではビルドできません。wasip1 のフルビルド (`-tags nosql,noopenpubkey`) は
  gzip で 11.5 MB です。CI には両方のバリアントをビルドする `wasip1` ジョブが
  あります。
- **待ち受け**はエンジンのソケット (`stealthrocket/net/wasip1`、WasmEdge の
  ソケット拡張) を通します。Go の `net.Listen` は wasip1 では使えません。
- **シグナル処理なし。** Go の wasip1 ランタイムはループで譲りながらシグナルを
  待つため、`signal.Notify` が 1 つあるだけでネットワークポーラーが飢餓状態になり、
  リスナーが何も accept しませんでした。終了はホストがモジュールを終わらせる
  ことで行います。
- **TLS** はコンテナの手前で終端します。そこまでの経路がプライベートな場合は
  `plaintext = true` を使ってください。

{{< callout type="info" >}}
**go-wasmcontainers プレイグラウンドの次のリリースで提供予定**

`fileshare` イメージと、go-fileshare エクスプローラー (v0.4.0) をコンテナの
ブラウザクライアントとして開く「explore :PORT」ボタン
([playground#31](https://github.com/go-wasmcontainers/playground/pull/31)、
保留中)。まだ利用できません。
{{< /callout >}}

## サブプロセスのプラグインを使わない理由 {#why-not-subprocess-plugins}

これは議論ではなく**計測**で決めました。`hashicorp/go-plugin` は gRPC と protobuf を持ち込み、
それだけで **13.2 MB** かかります。3 つのプロトコルとすべてのドライバーを含むこのバイナリ全体よりも
大きい値です。プラグインホストは、何も読み込む前からサイズが 2 倍になり、各プラグインのバイナリも
それぞれ gRPC を抱えることになります。

攻撃対象領域の観点でも、実行しないものについてはタグのほうが強力な手段です。
**一度もコンパイルされなかったコードには、サンドボックスの有無にかかわらず到達できません。**

タグが与え*ない*のは、**実際に動かす**プロトコル同士の分離と、再コンパイルせずにプロトコルを追加する手段です。
どちらも現実的な要件であり、どちらもバイナリを小さくするのではなく、プロセスの境界を支持する議論です。
それがこの同じバイナリを使う[プロトコルごとに 1 プロセス]({{< relref "/operations/isolation.md" >}})であり、
プラグインフレームワークではありません。
