---
title: "プロトコルごとに 1 プロセス"
weight: 20
description: "--isolate で各プロトコルを専用のプロセスで実行する方法と、それを安全に保つルール。"
tags: [運用, 分離]
---

```sh
fileshare --config /etc/fileshare.d --isolate
```

```
smb    on 0.0.0.0:445  — process 19810, serving public, photos and scratch
webdav on 0.0.0.0:8080 — process 19811, serving public and photos
nfs    on 0.0.0.0:2049 — process 19812, serving public
```

親プロセスがリスナーをバインドし、その後プロトコルごとに**自分自身**を exec して、
各子プロセスにはそのプロトコルが提供してよい共有だけを渡します。

## コードを読むのではなく、`lsof` で確認する {#checked-with-lsof-not-by-reading-the-code}

```
smb     (pid 19810) has open: scratch.img photos.img
webdav  (pid 19811) has open: photos.img
nfs     (pid 19812) has open: photos.img
```

書き込み可能なイメージが開かれているのは**ちょうど 1 つのプロセス**だけで、WebDAV の子プロセスは
それをまったく開きません。

これは[ビルドタグ]({{< relref "/operations/build-tags.md" >}})では得られないものです。あるプロトコルでの
パニックやヒープの枯渇が影響するのは、そのプロトコルだけです。各子プロセスはオペレーティングシステムが
提供するあらゆる手段で隔離でき、あるパーサーのバグが、そのプロセスが一度も開いていない
イメージに到達することはありません。

## 特権ポート {#privileged-ports}

Unix では、親プロセスがバインド済みのリスナーを子プロセスに渡すので、**特権ポートも特権のない
子プロセスで使えます**。親プロセスが 445 をバインドし、子プロセスは特権を必要としません。

Windows には `ExtraFiles` がないため、子プロセスが自分でアドレスをバインドします。分離は同じですが、
特権ポートに関する利点は得られません。

## 誠実さを保つルール {#the-rule-that-makes-it-honest}

子プロセスは自分でイメージを開くため、2 つのプロトコルで*書き込み可能*として提供されるイメージは、
1 つのファイルに対する 2 つのドライバーとなり、その間に**ロックがありません**。これはまさに、
1 つのプロセスの内側で[共有ロック]({{< relref "/operations/locking.md" >}})が防いでいるものです。

これは拒否され、拒否メッセージには修正方法が示されます。

```
scratch is writable over smb and webdav, and one process per protocol means
that many drivers writing one file with no lock between them. Say
`protocols = ["smb"]` on the share, or make it read_only, or do not isolate.
```

共有の `protocols = [...]` は、`--isolate` のときだけでなく単独でも有用です。SMB 専用と宣言した
共有は、WebDAV プロセスには一切通知されません。結果として何も提供しなくなる `serve` ブロックも、
イメージを開く前に拒否され、そこから除外された共有とその理由が示されます。

## `admin` ブロックや `metrics` ブロックとは、まだ併用できません {#not-with-an-admin-or-a-metrics-block-yet}

`--isolate` を [`admin`]({{< relref "/administration/_index.md" >}}) ブロックや
[`metrics`]({{< relref "/administration/health.md" >}}) ブロックと併用することは拒否されます。
イメージを開くのは子プロセスで、親プロセスは何も開かないため、API の変更を適用できる単一のプロセスも、
プローブが準備状態を尋ねる単一のプロセスも存在しないからです。
