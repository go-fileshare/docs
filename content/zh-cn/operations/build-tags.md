---
title: "只构建所需内容"
weight: 10
description: "将协议、用户目录服务、管理 API 和配给程序排除在二进制文件之外的构建标签，以及各自能节省多少。"
tags: [运维, 构建]
---

每个协议都位于一个构建标签之后，使用标签会将其**完全**排除：没有
监听器、没有解析器、没有依赖、没有代码。

```sh
go install -tags nonfs,nowebdav,nosftp,nos3 github.com/go-fileshare/fileshare@latest   # 仅 SMB
go build   -tags nosmb,nonfs,nosftp,nos3 .                                            # 仅 WebDAV
```

**人员**的来源也有各自的标签：`nosql` 排除
三个数据库驱动，`noldap` 排除 LDAP 客户端。

| 构建 | 大小 |
|---|---|
| 全部 | 31.2 MB |
| `-tags noldap` | 30.9 MB |
| `-tags nosftp` | 30.6 MB |
| `-tags noopenpubkey`（SFTP 上没有 opkssh 登录） | 约少 2 MB |
| `-tags nos3` | 31.1 MB |
| `-tags nonfs,nowebdav,nosftp,nos3`（仅 SMB） | 28.1 MB |
| `-tags nopartitioned`（没有 apfs、btrfs、xfs、zfs） | 29.2 MB |
| `-tags nosql` | 20.2 MB |
| `-tags nosql,noldap` | 19.8 MB |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp,nos3` | 16.2 MB |

这些是 fileshare README 给出的数字，测量于管理
API 出现之前；下面的表格是后来的测量结果。

## 管理 API 与 `nogrpc` {#the-admin-api-and-nogrpc}

[管理 API]({{< relref "/administration/_index.md" >}})引入了 gRPC 和 protobuf，
它们位于 `nogrpc` 之后（linux/amd64，测量于 2026-09-29，当时上面各行
的“全部”已增长到 34.1 MB）：

| 构建 | 大小 |
|---|---|
| 全部，含管理 API | 46.1 MB |
| `-tags nogrpc` | 34.4 MB |
| `-tags nosql,noldap,nogrpc` | 22.5 MB |

gRPC 的代价是 **11.7 MB**，即下面的[插件测量](#why-not-subprocess-plugins)
在这里因其自身的理由而再次进行的结果。通过文件管理共享的站点
不需要携带它。

{{< callout type="warning" >}}
**在 `nogrpc` 构建中，`admin` 块会被拒绝**

而不是在没有管理 API 的情况下提供服务：请求 API 却在没有它的情况下
启动的配置，并没有按其所说的去做。
{{< /callout >}}

`nogrpc` 会排除管理 API，同时也排除
[配给程序]({{< relref "/administration/volumes.md" >}})。[健康检查与指标]({{< relref "/administration/health.md" >}})
是普通的 HTTP，予以保留。

`-tags noprovisioner` 只排除配给程序，即特权的
`fileshare provisioner` 角色，以及随之而来的 go-fsctl 的 zfs、btrfs 和 projquota：
从不提供卷服务的服务器无需携带创建卷的代码。
这种构建中的 `fileshare provisioner` 会说明原因，而不是显示“unknown command”。

`nosql` 是迄今为止最大的杠杆：PostgreSQL、MySQL 和 SQLite
合计重 **11.7 MB**，超过本程序中所有协议加在一起的大小。
用户都在文件中的站点需要它。这些驱动由命令导入，
而不是由使用它们的库导入，这使得这一选择成为构建标签，
而不是分叉。

{{< callout type="info" >}}
**此二进制文件构建时未包含的协议**

指明它的配置会被告知*这一点*，而不是“没有这样的
协议”，这就是拼写错误与构建标签之间的区别。`noopenpubkey` 下的
`opkssh_client_id` 和 `nogrpc` 下的 `admin` 块也是如此。
{{< /callout >}}

## 为什么不使用子进程插件 {#why-not-subprocess-plugins}

这是**经过测量而不是争论出来的**：`hashicorp/go-plugin` 会引入 gRPC 和
protobuf，仅它们就占 **13.2 MB**，比包含全部三个协议和所有驱动的整个
二进制文件还大。插件宿主在加载任何东西之前就已是其两倍大小，
而每个插件二进制文件又会再次携带 gRPC。

就攻击面而言，对于任何你不运行的内容，标签也是更强的工具：
**从未编译的代码无法被触及**，无论是否有沙箱。

标签*无法*提供的是你**确实**运行的各协议之间的隔离，也无法
提供不重新编译就添加协议的方式。这两点都是真实存在的，而且它们都是
支持采用进程边界、而不是更小二进制文件的理由，也就是
[每个协议一个进程]({{< relref "/operations/isolation.md" >}})，使用的是同一个二进制文件，而不是插件
框架。
