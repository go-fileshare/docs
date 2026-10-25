---
title: "運用"
weight: 50
description: "バイナリのビルド方法、プロトコルの実行方法、そしてイメージを共有する方法。"
tags: [運用]
---

{{< cards >}}
  {{< card link="build-tags/" title="必要なものだけをビルドする" icon="cube" subtitle="ビルドタグを指定すると、プロトコルや管理 API をバイナリから完全に除外できます。" >}}
  {{< card link="isolation/" title="プロトコルごとに 1 プロセス" icon="view-grid" subtitle="--isolate: 各プロトコルを専用のプロセスで実行します。" >}}
  {{< card link="locking/" title="1 つのイメージ、複数のプロトコル、1 つのロック" icon="lock-closed" subtitle="複数のプロトコルから 1 つのイメージへの書き込みは、共有の 1 つのロックを通ります。" >}}
{{< /cards >}}
