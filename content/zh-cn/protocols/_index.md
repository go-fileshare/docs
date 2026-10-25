---
title: "协议能够承诺什么"
linkTitle: "协议"
weight: 20
description: "为什么每个协议能或不能分辨是谁在请求，以及这对它提供哪些共享意味着什么。"
tags: [协议]
---

各协议在访问控制所需要的一件事上并不一致：服务器能否分辨
**是谁**在请求。

| | |
|---|---|
| **[SMB]({{< relref "/protocols/smb.md" >}})** | NTLMv2。密码从不在网络上传输，而且共享会在读者尝试之前，通过访问掩码告知读者其权限。 |
| **[WebDAV]({{< relref "/protocols/webdav.md" >}})** | 基于 [TLS]({{< relref "/security/tls.md" >}}) 的 HTTP Basic：在可访问的地址上拒绝明文，除非 `plaintext = true` 表明有代理在前端终结 TLS。对于某人无权使用的共享，返回 404 而不是 403：不确认它是否存在。 |
| **[SFTP]({{< relref "/protocols/sftp.md" >}})** | **公钥**，或来自你所信任的颁发机构的 **SSH 证书**：服务器从不持有机密，使用证书时，一个人的访问权限在别处签发并在别处到期。没有密码：提示输入密码的客户端正在做密钥本来要避免的事情。 |
| **[OIDC]({{< relref "/protocols/webdav.md" >}})**（通过 WebDAV） | 由身份提供方签名的**持有者令牌**。由 [go-authn/oidc](https://github.com/go-authn/oidc) 验证：签名、颁发者、受众、有效期。这里没有其他协议能存放令牌：S3 使用 SigV4 签名，没有可容纳持有者令牌的字段。通过 [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})时，提供方的凭证则随证书到达。 |
| **[S3]({{< relref "/protocols/s3.md" >}})** | **SigV4**，标头或预签名方式。机密通过计算 HMAC 来证明自己，且从不在网络上传输，因此与 NTLMv2 一样，目录服务必须持有密码，而不仅仅是检查密码。共享就是一个存储桶。 |
| **[NFSv3]({{< relref "/protocols/nfs.md" >}})** | 其自身**什么都没有**。`AUTH_UNIX` 只是一种声明：客户端说“uid 501”，而网络无法反驳。`kerberos` 块可以解决这一点，[基于证书的身份]({{< relref "/security/nfs-certificates.md" >}})也可以。[TLS]({{< relref "/security/tls.md" >}})会对其加密，并证明的是机器，而不是个人。 |

## 一次性说明后果 {#the-consequence-stated-once}

**指明了谁可以使用的共享不会通过 NFS 导出**，除非
配置了 Kerberos，或者 NFS 采用
[基于证书的身份]({{< relref "/security/nfs-certificates.md" >}})。

这不是警告，也不是选项：一个声明 *photos 属于 alice* 的配置，
和一个把 photos 交给任何连接者的协议，不可能同时得到遵守，
而悄悄地扩大访问范围是两种失败中更糟的那一种。拒绝信息会
在启动时以及 [`check`]({{< relref "/configuration/check.md" >}})中打印，并附上原因。

## 每个协议对目录服务有什么要求 {#what-each-one-needs-from-a-directory}

哪些协议可以为某个人提供服务，取决于其来源能够证明什么，
而不是仅由配置决定。参见
[来源能够证明什么]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}})。

## 哪些是加密的 {#what-is-encrypted}

WebDAV、S3 和 NFS 接受 `tls = true`；SMB 3 使用自己的密钥加密，而 SFTP
本身就是 SSH。参见 [TLS 与来自 ACME 的证书]({{< relref "/security/tls.md" >}})。
