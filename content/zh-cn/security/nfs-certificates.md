---
title: "NFS 与基于证书的身份"
weight: 30
description: "向由 X.509 客户端证书指明的人员，通过 NFS 提供受限共享，并由 CRL 撤销，以及在共享客户端上它无法承诺什么。"
tags: [安全, nfs, tls, 撤销]
---

NFSv3 自身不对任何人进行身份验证，这就是指明了谁可以使用的共享
[不会通过 NFS 导出]({{< relref "/protocols/nfs.md" >}})的原因，除非有东西能够
区分不同的人。`kerberos` 块是一种答案。自 v0.12.0 起，
**指明某个人**的客户端证书是另一种：

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # 或 crl_file；必须给出其一
  crl_state_file = "/var/lib/fileshare/nfs.crl.state"     # 跨重启保持先后顺序
}
```

## 证书携带什么 {#what-the-certificate-carries}

在身份提供方登录之后，go-authn/bridge 会签发一份**短期有效的 X.509
客户端证书**，以 FreeBSD 的 `rpc.tlsservd -u` 读取的方式指明
此人：

- 此人：SubjectAltName **otherName `1.3.6.1.4.1.2238.1.1.1`**，一个
  UTF8String `user@domain`
  （[draft-cel-nfsv4-rpc-tls-othername](https://www.ietf.org/archive/id/draft-cel-nfsv4-rpc-tls-othername-04.html)
  对它作了描述；其 OID 尚未分配）；
- 此人所属的组：每个组一个 SubjectAltName URI，
  `tag:go-authn.github.io,2026:group:<group>`，即 **tag URI**（RFC 4151：无需
  注册，正是为此而设计）。FreeBSD 的 `rpc.tlsservd` 读取第一个
  身份 otherName，并跳过其他所有 SAN 条目。

{{< callout type="info" >}}
**为什么用 tag URI，而不是自己的扩展**

曾经设计过一个，使用由 UUID 派生的弧（`2.25.<128 位数字>`），并经过**实测**：
Go 的 `x509.ParseCertificate` 会拒绝整个证书，提示
*malformed extension OID field*，因为 ASN.1 对象标识符的弧在那里是
`int`。每个 Go TLS 服务器都会拒绝这份证书。
URI SAN 不需要 OID，每个解析器都能读取它。也不使用 `urn:x-…`：
RFC 8141 取消了实验性的 URN 命名空间，严格的解析器
会拒绝它。
{{< /callout >}}

## 每次调用都会被检查什么 {#what-every-call-is-asked}

设置 `identity = "certificate"` 后，指明了人员的共享可以通过 NFS 提供服务，
**无需 kerberos**。针对它的每次调用都必须通过 TLS 到达，并且其证书必须：

1. **不在 CRL 中**；
2. 恰好指明**一个**人；
3. 指明的人是本服务器以**对待令牌的方式**准入的：规则、
   `trust_all`、`domains`，参见
   [由身份提供方指明的人员]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})；
4. 指明的人是**共享所允许的**。

并且，如果有 [`ssf` 块]({{< relref "/security/shared-signals.md" >}})，证书必须是在对此人的任何撤销**之后**
签发的（以其 `NotBefore` 为准）。

## 撤销：CRL {#revocation-the-crl}

**CRL 是必需的**，即 `crl_url` 或 `crl_file`。无法被任何东西撤销的
个人证书，会在此人被移除之后继续存在整个有效期。

| 字段 | 默认值 | |
|---|---|---|
| `crl_url` | | 仅限 HTTPS |
| `crl_file` | | 或绝对路径，二者取其一 |
| `crl_ca_file` | 系统的 | 固定 CRL 的 HTTPS 服务器的颁发机构 |
| `crl_refresh` | `1m` | 获取的频率 |
| `crl_max_age` | `1h` | 最近一份有效副本最多可以有多旧 |
| `crl_state_file` | | 保存最近一份通过验证的 CRL 的位置，以便重启后记得哪一份更新 |

它的获取和保存方式与 [KRL]({{< relref "/security/revocation-lists.md" >}})相同，并同样**以拒绝为默认**；
超过自身 **`NextUpdate`** 的 CRL 无论 `crl_max_age` 允许与否都视为未知。CRL 必须由客户端 CA 签名：任何人
都可以编写的 CRL 可以随意撤销，更糟的是，也可以撤销后又恢复。只要
CRL 可访问，**它变化之后的下一次调用就会被拒绝**；只有在 CRL 无法访问时，
撤销才受证书有效期的限制。

自 v0.16.2 起，会通过 [go-authn/revocation](https://github.com/go-authn/revocation)
在解析 CRL **之前**对原始字节检查签名：
如果先解析，由任何人签名的大型 CRL 在被拒绝之前就会消耗内存和数秒时间。
本服务器无法读作完整列表的内容同样会被拒绝：
增量 CRL、未知的关键扩展或条目扩展、没有 CRL
编号、没有 `NextUpdate`。编号低于已持有 CRL 的 CRL，或编号相同但签发更早的 CRL，
会像更旧的
[KRL]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}})一样被拒绝。

没有 `identity = "certificate"` 的 CRL 字段会被拒绝，没有 `tls = true` 和
`client_ca_file` 的 `identity = "certificate"` 同样如此：
身份就在客户端出示的证书中。

## 它无法承诺什么 {#what-it-cannot-promise}

{{< callout type="error" >}}
**Linux 客户端的证书属于一次挂载，而不是一个人**

服务器看到的是**连接**的证书，而 Linux 客户端把证书
附加到挂载上（`tlshd`，`mount -o xprtsec=mtls,...` 上的
密钥环序列号）。**该挂载的每个用户都以证书所指明的那个人的身份行事。**
对于一个人使用的工作站，这恰好就是
此人；对于多人登录的机器，则是挂载者，请在那里使用
`sec=krb5`。

这就是 RFC 9289 本身拒绝承诺用户身份验证的原因，也是普通的
[基于 TLS 的 NFS]({{< relref "/security/tls.md#nfs-over-tls-proves-the-machine-not-the-person" >}})
仍然只能证明机器的原因。
{{< /callout >}}

## 用真实 Linux 客户端实测 {#measured-with-a-real-linux-client}

内核 6.17，ktls-utils 0.9，在 go-filesystems/nfs 的 CI 中：

- **MOUNT 的 `MNT` 以明文到达**，无论 `xprtsec=` 如何设置：内核的
  挂载客户端不支持 TLS。它会得到应答，而随后每个未使用 TLS 的 NFS 调用
  都会被拒绝，因此共享不允许的人在第一次访问时就会看到
  *access denied*，而不是在 `mount` 时。**共享的任何内容都不会以明文传输。**
- `tlshd` 只对照**系统**信任库检查本服务器的证书，
  它会忽略 `x509.truststore`，并且要求客户端的证书和
  密钥**归 root 所有，密钥模式为 600**。否则它会以
  *gnutls: Error in the certificate (-43)* 失败，两者都不会指明。
