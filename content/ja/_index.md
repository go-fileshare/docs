---
title: "go-fileshare"
linkTitle: "ホーム"
type: docs
cascade:
  type: docs
description: "go-fileshare は、ディスクイメージとディレクトリを SMB、NFS、WebDAV、SFTP、S3 で提供します。ユーザーと共有ごとのルールは、1 つの設定から一元的に管理されます。"
---

**ディスクイメージ(またはディレクトリ)を SMB、NFS、WebDAV、SFTP、S3 で提供します。
同じユーザー、同じ共有ごとのアクセス権を、1 つの設定ファイルから管理できます。** 純粋な
Go 製で、`CGO_ENABLED=0`、バイナリは 1 つです。このページは
[v0.31.2]({{< relref "/status.md" >}}) について説明しています。

```sh
go install github.com/go-fileshare/fileshare@latest          # with Go 1.27.3 or later

fileshare --image disk.img --user alice --password-file pw   # one image, now
fileshare --config /etc/fileshare.d                          # several, with users
```

**Go が一切不要なバイナリ**も利用できます。v0.29.1 以降のすべてのリリースには、プラットフォームごとの
静的バイナリが[リリースページ](https://github.com/go-fileshare/fileshare/releases)に用意されています
(Linux は 6 つのアーキテクチャ、macOS、Windows、FreeBSD、OpenBSD、そして v0.29.2 以降は NetBSD)。
`sha256sum -c --ignore-missing SHA256SUMS` で検証し、さらに
`gh attestation verify <アーカイブ> -R go-fileshare/fileshare` を実行すると、
GitHub がそのタグからビルドしたことを確認できます。

パスワードは**ファイル**から読み込み、フラグでは渡しません。引数はプロセス一覧に表示され、
同じマシン上のすべてのユーザーから見えてしまうためです。

## なぜ 1 つのプログラムなのか {#why-one-program}

イメージをどのプロトコルで運ぶかは、相手側のクライアントによって決まります。
macOS と Windows は SMB、Linux のサーバー群にはすでに NFS があり、ブラウザーや
スマートフォンは HTTP を使います。3 つのサーバーを、それぞれ別の設定ファイルと
`alice` が誰であるかについての別々の解釈で運用すると、そのうち 2 つで微妙な誤りを招きがちです。

そこで、設定は 1 つ、ユーザーの集合は 1 つ、共有ごとのルールの集合も 1 つとし、
各プロトコルはその上のリスナーとして動作します。

## 起動時に出力される内容 {#what-it-prints-at-startup}

```
smb    on 0.0.0.0:445 — photos and scratch
webdav on 0.0.0.0:8080 — photos and scratch
sftp   on 0.0.0.0:2222 — photos and scratch
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

この最後の段落が、プログラム全体の方針を表しています。設定が求めるルールと、それを守れない
プロトコルとが、黙って妥協点で折り合うことはありません。
[プロトコルが保証できること]({{< relref "/protocols/_index.md" >}})を参照してください。

## 次に読むページ {#where-to-go-next}

| | |
|---|---|
| [設定]({{< relref "/configuration/_index.md" >}}) | ユーザー、グループ、共有、`serve` ブロック |
| [共有: イメージ、デバイス、ディレクトリ]({{< relref "/configuration/shares.md" >}}) | 自動検出されるもの、宣言が必要なもの |
| [ユーザー、グループ、ディレクトリ]({{< relref "/configuration/identity.md" >}}) | ファイル、SQL、LDAP、そしてそれぞれが証明できること |
| [`check`]({{< relref "/configuration/check.md" >}}) | 再起動の前に設定全体を読み戻して確認する |
| [プロトコル]({{< relref "/protocols/_index.md" >}}) | それぞれが保証できること、できないこと |
| [管理 API]({{< relref "/administration/_index.md" >}}) | 再起動なしで共有の作成、権限の付与、オフライン化を行う |
| [ボリューム]({{< relref "/administration/volumes.md" >}}) | ZFS データセット、btrfs サブボリューム、XFS/ext4 プロジェクトクォータを API で作成して提供する |
| [ヘルスチェックとメトリクス]({{< relref "/administration/health.md" >}}) | `/healthz`、`/readyz`、`/metrics` |
| [ディレクトリの再読み込み]({{< relref "/administration/reload.md" >}}) | `reload`、`SIGHUP`、そして削除が開いているセッションに与える影響 |
| [TLS]({{< relref "/security/tls.md" >}}) | ファイルまたは ACME、そして WebDAV を平文で拒否する理由 |
| [何が何を失効させるか]({{< relref "/security/_index.md" >}}) | KRL、CRL、共有シグナル、そしてそれぞれがフェイルクローズする理由 |
| [必要なものだけをビルドする]({{< relref "/operations/build-tags.md" >}}) | タグを指定するとプロトコルを丸ごと除外できる |
| [プロトコルごとに 1 プロセス]({{< relref "/operations/isolation.md" >}}) | `--isolate` |
| [ステータス]({{< relref "/status.md" >}}) | 検証済みのもの、まだ存在しないもの |

## ライセンス {#licence}

BSD-3-Clause。
