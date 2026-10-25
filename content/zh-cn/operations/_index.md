---
title: "运维"
weight: 50
description: "二进制文件如何构建，各协议如何运行，以及它们如何共享同一个映像。"
tags: [运维]
---

{{< cards >}}
  {{< card link="build-tags/" title="只构建所需内容" icon="cube" subtitle="通过构建标签，可将某个协议或管理 API 完全排除在二进制文件之外。" >}}
  {{< card link="isolation/" title="每个协议一个进程" icon="view-grid" subtitle="--isolate：每个协议运行在各自的进程中。" >}}
  {{< card link="locking/" title="一个映像，多个协议，一把锁" icon="lock-closed" subtitle="多个协议对同一映像的写入都经过同一把共享锁。" >}}
{{< /cards >}}
