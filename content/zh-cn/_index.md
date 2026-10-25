---
title: "go-fileshare"
linkTitle: "首页"
type: docs
cascade:
  type: docs
description: "go-fileshare 通过 SMB、NFS、WebDAV、SFTP 和 S3 提供磁盘映像和目录服务，使用同一套用户，并由同一份配置给出各共享的规则。"
---

**一个磁盘映像，或一个目录，通过 SMB、NFS、WebDAV、SFTP 和 S3 提供服务：
相同的用户，相同的按共享访问控制，来自同一个配置文件。** 纯
Go 实现，`CGO_ENABLED=0`，单一二进制文件。本文档描述的是
[v0.30.0]({{< relref "/status.md" >}})。

```sh
go install github.com/go-fileshare/fileshare@latest

fileshare --image disk.img --user alice --password-file pw   # 一个映像，立即可用
fileshare --config /etc/fileshare.d                          # 多个映像，带用户
```

也可以直接使用**二进制文件，完全不需要 Go**：自 v0.29.1 起，每个发布版本都在其
[发布页面](https://github.com/go-fileshare/fileshare/releases)提供各平台的静态二进制文件
（Linux 六种架构、macOS、Windows、FreeBSD、OpenBSD，以及自 v0.29.2 起的 NetBSD）。
请使用 `sha256sum -c --ignore-missing SHA256SUMS` 进行校验，并使用
`gh attestation verify <archive> -R go-fileshare/fileshare` 确认该文件由
GitHub 根据对应标签构建。

密码来自**文件**，绝不通过命令行选项传入：命令行参数对机器上的所有用户
在进程列表中都是可见的。

## 为什么只用一个程序 {#why-one-program}

用哪种协议传输映像，取决于另一端的客户端：
macOS 和 Windows 倾向于 SMB，Linux 集群已经有 NFS，浏览器或手机
使用 HTTP。运行三个服务器，每个都有各自的配置文件，
各自对 `alice` 是谁有不同的理解，这很容易让其中两个出现细微的错误。

因此这里只有一份配置、一套用户、一套按共享的规则，
各协议只是建立在其上的监听器。

## 启动时的输出 {#what-it-prints-at-startup}

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

最后这一段体现了整个程序的特点：配置所要求的规则与无法满足它的协议，
不会悄悄地各让一步。
参见[协议能够承诺什么]({{< relref "/protocols/_index.md" >}})。

## 接下来读什么 {#where-to-go-next}

| | |
|---|---|
| [配置]({{< relref "/configuration/_index.md" >}}) | 用户、组、共享、`serve` 块 |
| [共享：映像、设备、目录]({{< relref "/configuration/shares.md" >}}) | 哪些会被自动检测，哪些必须声明 |
| [用户、组和目录服务]({{< relref "/configuration/identity.md" >}}) | 文件、SQL、LDAP，以及各自能够证明什么 |
| [`check`]({{< relref "/configuration/check.md" >}}) | 重启之前完整回读整个配置 |
| [协议]({{< relref "/protocols/_index.md" >}}) | 各协议能够承诺什么、不能承诺什么 |
| [管理 API]({{< relref "/administration/_index.md" >}}) | 无需重启即可创建共享、授权共享和将共享下线 |
| [卷]({{< relref "/administration/volumes.md" >}}) | ZFS 数据集、btrfs 子卷以及 XFS/ext4 项目配额，通过 API 创建并提供服务 |
| [健康检查与指标]({{< relref "/administration/health.md" >}}) | `/healthz`、`/readyz`、`/metrics` |
| [重新读取目录服务]({{< relref "/administration/reload.md" >}}) | `reload`、`SIGHUP`，以及移除操作对已打开会话的影响 |
| [TLS]({{< relref "/security/tls.md" >}}) | 文件或 ACME；以及为什么拒绝明文 WebDAV |
| [什么撤销什么]({{< relref "/security/_index.md" >}}) | KRL、CRL、共享信号，以及为什么它们都以拒绝为默认 |
| [只构建所需内容]({{< relref "/operations/build-tags.md" >}}) | 通过构建标签完全省略某个协议 |
| [每个协议一个进程]({{< relref "/operations/isolation.md" >}}) | `--isolate` |
| [状态]({{< relref "/status.md" >}}) | 哪些已经验证，哪些尚未实现 |

## 许可证 {#licence}

BSD-3-Clause。
