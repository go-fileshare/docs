---
title: "撤销列表无法涵盖的内容：共享信号"
linkTitle: "共享信号"
weight: 40
description: "通过来自共享信号发送方的 CAEP session-revoked 事件，撤销访问令牌、OpenPubkey 证书和其他联盟凭据。"
tags: [安全, 撤销, 共享信号]
---

证书有撤销列表，即 [KRL]({{< relref "/security/revocation-lists.md" >}})和
[CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})。而 fileshare 自行验证的
**访问令牌**，以及由个人自己的密钥签名的 **OpenPubkey 证书**则没有：
它们在到期之前一直有效，无论此人之后发生了什么。

自 v0.13.0 起，[OpenID 共享信号框架](https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html)
通过 [CAEP](https://openid.net/specs/openid-caep-1_0-final.html)
**`session-revoked`** 事件弥补了这一点：

```hcl
ssf {
  transmitter        = "https://bridge.example.org"          # 它的颁发者
  audience           = "https://files.example.org"           # 本服务器在它眼中的名称
  client_id          = "fileshare"                           # OAuth 客户端凭据，
  client_secret_file = "/etc/fileshare/ssf.secret"           # 作用域为 "ssf"（或 token_file）
  state_file         = "/var/lib/fileshare/revocations.json"
  max_age            = "10m"                                 # 默认值；retain = "192h"
}
```

go-authn/bridge 在停用个人或 IdP 时会发送一个；fileshare
对其进行**轮询**（RFC 8936，SSF 的默认方式），并为每个人记录它
**何时**发生。

## 此前签发的一切，在所有协议上都会被拒绝 {#everything-issued-before-is-refused-on-every-protocol}

从那一刻起，**此前签发**的每份联盟凭据都会被拒绝，
**它们所打开的会话也不再获得服务**，经过的关卡与
KRL 相同。“签发”的含义取决于由什么承载它：

| 协议 | 凭据 | 其签发时间 |
|---|---|---|
| WebDAV | 访问令牌 | 其 `iat` |
| SFTP | 由提供方的 SSH CA 签发的证书 | 其有效期起点（`ValidAfter`） |
| SFTP | OpenPubkey（opkssh）证书 | ID 令牌的 `iat` |
| NFS | 客户端证书（[identity = "certificate"]({{< relref "/security/nfs-certificates.md" >}})） | 其 `NotBefore` |

提供方在**此后**签发的凭据则属于有效凭据：重新启用的人
不会被锁在门外。

{{< callout type="info" >}}
**同一秒算作之前**

CAEP 的 `event_timestamp` 和令牌的 `iat` 都是整秒，而在撤销
**当时或之前**签发的凭据会被拒绝，因此与撤销在同一秒
签发的凭据同样被拒绝，刚被停用后立即重新启用的人
从下一秒起获得可用的凭据。
这是有意选择的保守一侧。未知的签发时间
并不等于“很久以前”：只要有任何针对该人的撤销，
它就会被拒绝。
{{< /callout >}}

## 事件针对的是谁 {#who-an-event-is-about}

主体是 RFC 9493 的 **`account`**（`acct:user@domain`，即共享所使用的名称）、
**`iss_sub`**、**`email`**，或它们的 **`aliases`**。

**整个被停用的 IdP** 以 CAEP *tenant* 主体的形式到达，事件中带有 IdP 的
作用域：名称为 `@` 加其中之一（整体比较，
`evil-univ-a.fr` 不是 `univ-a.fr`）、且凭据在此前签发的每个人，
都会被拒绝，**包括提供方已不再记得的人**。

## 接收方如何进行身份验证 {#how-the-receiver-authenticates}

使用 **OAuth 客户端凭据**（RFC 6749 §4.4，作用域 `ssf`）：`client_id` 和
`client_secret_file`，令牌从发送方自己的令牌端点获取，该端点
可在其 OpenID 配置中找到，或由 `token_url` 指定，并在其过期之前续期。
`token_file` 则用于发放长期持有者令牌的发送方。二者取其一：
两者都给出或都不给出，都会被拒绝。

| 字段 | 默认值 | |
|---|---|---|
| `transmitter` | | 其颁发者，**仅限 HTTPS** |
| `audience` | | 本服务器在发送方眼中的名称；必填，否则发给其他接收方的 SET 会在这里被信任 |
| `client_id`、`client_secret_file` | | OAuth 客户端凭据，必须同时给出 |
| `token_url` | 通过发现获得 | 仅限 HTTPS，否则机密会以明文穿过网络 |
| `token_file` | | 用长期令牌代替客户端凭据 |
| `state_file` | | **必填**：记录撤销的位置 |
| `ca_file` | 系统的 | 固定发送方的颁发机构 |
| `max_age` | `10m` | 在拒绝联盟凭据之前，多久没有收到发送方的消息；至少一分钟 |
| `retain` | `192h` | 撤销记录保留多久；至少 169h（自 v0.14.0 起） |

撤销**在被确认之前就已记录**，因此已确认的撤销
在重启后依然有效；并且它会保留 `retain` 的时间，长于它
可能使之失效的任何凭据（192h 超过提供方签发的最长寿命凭据，即
168h 的 SSH 证书）。

没有任何联盟凭据可供撤销的 `ssf` 块（没有 `oidc` 块，也没有 NFS
`identity = "certificate"`）会被拒绝。

传输层使用 [github.com/hstern/go-ssf](https://github.com/hstern/go-ssf)：
发现、RFC 8936 轮询器、SET 的 JWS 层。事件的**含义**则由
fileshare 决定。

## 它以拒绝为默认 {#it-fails-closed}

{{< callout type="error" >}}
**在发送方沉默期间，联盟凭据会被拒绝**

与 KRL 一样：在发送方没有在 `max_age` 内应答轮询期间，
**每份联盟凭据**都会被拒绝，因为“没有撤销到达”和“撤销无法到达”
看起来是一样的。
`fileshare_ssf_last_heard_seconds` 是应当设置告警的指标；
`fileshare_ssf_revoked_subjects` 是保留的撤销数量。参见
[健康检查与指标]({{< relref "/administration/health.md#what-to-alert-on" >}})。
{{< /callout >}}

## 端到端检查 {#checked-end-to-end}

go-authn/bridge 的互操作通道
（[#12](https://github.com/go-authn/bridge/pull/12)、
[#17](https://github.com/go-authn/bridge/pull/17)）针对真实的 bridge 评判 fileshare v0.13.0：
KRL、基于 NFS 的 CRL、通过 SSF 撤销的 WebDAV 令牌、按作用域停用的 IdP，
以及基于 SSF 的 opkssh，这是任何撤销列表都无法触及的情形。
