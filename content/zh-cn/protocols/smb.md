---
title: "SMB：NTLMv2"
linkTitle: "SMB：NTLMv2"
weight: 10
description: "使用 NTLMv2 的 SMB：目录服务必须持有什么、隔离模式下的特权端口，以及目录服务重新加载对已打开连接的影响。"
tags: [协议, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

密码**从不在网络上传输**：客户端发送的是由密码计算出的证明。
共享会在读者尝试之前，通过访问掩码告知读者其权限，
而不是让其稍后因写入失败才发现。

## 目录服务必须持有什么 {#what-a-directory-must-hold}

NTLMv2 需要密码本身或其 MD4（`sambaNTPassword`，即 `nt_hash`
列）。**其他任何东西都不行。**

只能*检查*密码的目录服务，例如 LDAP 绑定或 bcrypt 列，
无法应答 SMB，无论检查有多好，因为服务器必须计算出
与客户端相同的证明。这是协议的属性，而不是本程序的
限制。

对于这样的人员，[`check`]({{< relref "/configuration/check.md" >}})会在配置阶段于 SMB 列中打印 `-`，
而不是让其在挂载时才发现。

## 无需特权服务器的特权端口 {#a-privileged-port-without-a-privileged-server}

在 Unix 上使用 [`--isolate`]({{< relref "/operations/isolation.md" >}}) 时，父进程绑定 445，
并把监听器传给子进程，因此真正使用 SMB 的进程
从不需要该特权。Windows 没有 `ExtraFiles`，因此在那里由子进程
自行绑定地址；隔离效果相同，但特权端口这一半则不然。

## 已对照本项目未编写的客户端进行验证 {#verified-against-a-client-this-project-did-not-write}

[go-smb2](https://github.com/cloudsoda/go-smb2) 在 CI 中的 `-race` 下，
通过 SMB 驱动二十个并发读取，同时有二十个通过 WebDAV 进行。macOS
通过 SMB 挂载共享，同时 `curl` 通过 WebDAV 向同一个映像写入，
并读回 WebDAV 所写的内容。

## 当人员发生变化时 {#when-the-people-change}

SMB 在**共享添加时**就固定了谁可以连接到该共享，并在每次树连接时
检查一次。因此，[目录服务重新加载]({{< relref "/administration/reload.md" >}})
发现的新人会原地添加，包括 SMB 正在运行的服务器，不触及任何连接；
但任何被收回的内容，无论是人员、凭据还是共享的展开
列表，都会产生新的世代，SMB 已打开的连接会被关闭，因此
撤销会触及已经打开的会话。`allow` 组已
清空的共享**根本不会通过 SMB 提供**：其空的 `AllowUsers` 会被理解为
所有人。
