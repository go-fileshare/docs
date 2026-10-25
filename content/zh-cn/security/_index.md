---
title: "什么撤销什么"
linkTitle: "安全"
weight: 40
description: "每种 fileshare 接受的凭据由哪种机制收回，以及为什么它们都以拒绝为默认。"
tags: [安全, 撤销]
---

凭据在出示时被检查。事后收回某个人的权限，例如离职的
员工、在 go-authn/bridge 中被停用的账户、整个被关闭的 IdP，
必须触及此人获得的**每一份**凭据，以及这些凭据已经打开的
会话。这里每种凭据都由不同的机制处理，因为每种凭据的签发和检查方式各不相同：

| 凭据 | 协议 | 由什么收回 | 引入版本 |
|---|---|---|---|
| `users` 目录服务（SQL、LDAP）中的密码、NT 哈希或应用密码 | SMB、WebDAV、S3 | 删除该行，然后[目录服务重新加载]({{< relref "/administration/reload.md" >}})：产生新世代，并关闭它所打开的连接 | v0.10.0 |
| 共享授权 | 所有协议 | [管理 API]({{< relref "/administration/_index.md" >}}) 的 `Revoke` 或 `DisableShare`：产生新世代，关闭连接 | v0.8.0 |
| 由提供方的 CA 签发的 SSH 证书 | SFTP | 其 [KRL]({{< relref "/security/revocation-lists.md" >}})，在登录时**以及每次操作时**检查；以及[共享信号]({{< relref "/security/shared-signals.md" >}}) | v0.12.0、v0.13.0 |
| OpenPubkey（opkssh）证书 | SFTP | 没有列表可以指明它；[共享信号]({{< relref "/security/shared-signals.md" >}})，以及 `opkssh_max_age` | v0.13.0 |
| 访问令牌 | WebDAV | 没有列表可以指明它；[共享信号]({{< relref "/security/shared-signals.md" >}}) | v0.13.0 |
| 指明某个人的 X.509 客户端证书 | NFS | 其 [CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})，在下一次调用时；以及[共享信号]({{< relref "/security/shared-signals.md" >}}) | v0.12.0、v0.13.0 |

没有第三列中的机制，凭据在到期之前一直有效；对于证书或令牌，
这个窗口期就是它的有效期。

## 它们每一个都以拒绝为默认 {#every-one-of-them-fails-closed}

无法获取的撤销列表、比其 `max_age` 更旧的副本、
超过自身 `NextUpdate` 的 CRL、`max_age` 之内没有收到消息的
共享信号发送方：这些都不会被理解为“没有任何东西被撤销”。
而是“本服务器不知道”，该机制所管辖的凭据会被
**拒绝**，直到它重新得知为止。

{{< callout type="error" >}}
**代价是可用性，而且是有意付出的**

在列表无法访问时放行一切的撤销检查，会被能够阻断该列表的
攻击者击败；这正是浏览器的软失败 OCSP 被称为“一旦撞车
就会断开的安全带”的原因。因此，列表或发送方
必须与它所管辖的登录一样可靠地提供服务，而表明它正在变旧的
指标，正是应当[设置告警]({{< relref "/administration/health.md#what-to-alert-on" >}})的指标。
{{< /callout >}}

只有解析成功的列表才会替换副本：到达时已损坏的列表会让
最近一份有效副本保持原位，并仍从**它**被获取的时刻起计入 `max_age`。
而且列表**只通过 HTTPS** 获取，因为路径上的攻击者可以替换的列表
什么都撤销不了，所以纯 HTTP 的 URL 会在
启动时被拒绝。

## 传输 {#transport}

凭据本身不得以明文穿过网络。这就是
[TLS]({{< relref "/security/tls.md" >}})：基于 TLS 的 WebDAV、S3 和 NFS、来自文件或 ACME 的证书，
并且自 v0.9.0 起，**带密码的 WebDAV 在明文下会被拒绝**。
