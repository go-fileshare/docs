---
title: "WebDAV：Basic 或持有者令牌"
linkTitle: "WebDAV：Basic 或持有者令牌"
weight: 20
description: "使用 HTTP Basic 或 OIDC 持有者令牌的 WebDAV、它所要求的 TLS，以及令牌在这里如何成为一个人。"
tags: [协议, webdav, oidc]
---

```hcl
tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true
}
```

HTTP Basic 或持有者令牌，都基于 [TLS]({{< relref "/security/tls.md" >}})，证书来自文件或
ACME。

{{< callout type="error" >}}
**自 v0.9.0 起不再允许明文**

HTTP Basic 在每个请求中就是密码本身，而持有者令牌与密码
等效。其他机器可以访问的地址上**没有 TLS** 的 WebDAV，
只要有人需要身份验证，就会**在启动时被拒绝**，
`serve "webdav" { addr = "0.0.0.0:8080" }`，即本页曾经
展示的示例，不再能够启动。如果有代理在前端终结 TLS，请明确说明：

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true
}
```

回环地址，以及没有任何人需要身份验证的服务器，不受影响。参见
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})。
{{< /callout >}}

{{< callout type="info" >}}
**是 404，而不是 403**

对于某人无权使用的共享，返回 **404**。不会确认它是否存在。
{{< /callout >}}

## 令牌，通过唯一能够承载它的协议 {#a-token-over-the-one-protocol-that-can-carry-one}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
}
```

浏览器有令牌而没有密码。因此 WebDAV 接受 `Authorization:
Bearer`，并且在也有本地人员的情况下，它发出的质询会同时提供
**两种**方式，由客户端选择它能够应答的那一种。令牌由
[go-authn/oidc](https://github.com/go-authn/oidc) 验证：签名、颁发者、受众、
有效期。

{{< callout type="error" >}}
**持有者令牌：仅限 WebDAV**

SMB 使用 NTLMv2 进行身份验证，SFTP 使用密钥或证书，NFS 什么都
没有，S3 使用 SigV4 签名：它们都没有地方放置
`Authorization` 标头。这是关于协议的事实，而不是本程序的
限制。

提供方的凭证改为通过证书到达 **SFTP**，参见
[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})。指明了提供方、却既不提供 WebDAV 也不提供此类 SFTP 的配置，
会**被拒绝，而不是启动**。
{{< /callout >}}

{{< callout type="error" >}}
**令牌说明的是提供方认为某人是谁，并不说明本服务器有属于此人的共享。**

对于这里没有任何来源认识的名称，即使令牌有效也会被拒绝：对
*我不认识你*的安全理解，不是*你被允许*。本地
账户同样拥有的名称，也不会因此获得该账户，除非 `local_names = true` 表示
提供方的名称就是本服务器的名称（自 v0.20.0 起默认关闭）；没有
该设置时，这样的令牌同样会被拒绝，除非有 `oidc:` 规则指明此人。
{{< /callout >}}

如果提供方**就是**目录服务，站点可以这样说明：

```hcl
oidc {
  issuer    = "https://login.example.org"
  audience  = "fileshare"
  trust_all = true      # 该提供方担保的所有人，而不仅仅是这里列出的人
}
```

这里**没有登录流程**：没有重定向，没有客户端机密，没有 cookie。
这是资源服务器。

令牌给出的**名称**是 `preferred_username`，或 `username_claim` 所指明的
声明。没有 `preferred_username` 时，只有在提供方声明 `email_verified: true` 的情况下
才使用电子邮件，否则使用 `sub`。设置 `username_claim = "email"` 时，
电子邮件未经验证的令牌会被拒绝，WebDAV 如此，
通过 SFTP 的 opkssh 登录也是如此。在验证之前，
电子邮件只是此人键入的内容，而携带它的已签名令牌
会绑定到一个无人核实过的地址（OpenID Connect Core 5.1；
go-authn/oidc v0.2.0，fileshare v0.16.3）。

提供方的哪些人可以获得共享（组、指明的个人、
整个机构）通过 `oidc:` 规则和 `domains` 来写；参见
[由身份提供方指明的人员]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})。

## 令牌比人更长寿，除非有东西另作说明 {#a-token-outlives-the-person-unless-something-says-otherwise}

令牌在这里独立验证，并在到期之前一直有效，无论
此人之后发生了什么。没有任何撤销列表可以指明它。
[`ssf` 块]({{< relref "/security/shared-signals.md" >}})会使提供方发出的 CAEP `session-revoked`
让此前（`iat`）签发的每个令牌失效。

## 根目录，以及分块上传 {#the-root-and-uploads-in-pieces}

**根目录会列出各共享**（自 v0.29.0 起）。浏览器访问 `/` 会得到一个包含
其可打开的共享的页面；对 `/` 的 `PROPFIND` 会将它们作为集合返回（`207`），
因此文件管理器可以挂载整个服务器，而不是单个共享。深度
`infinity` 会被拒绝（`403`，`propfind-finite-depth`，RFC 4918 允许这样做），
任何会在根目录写入的操作也会被拒绝（`405`）。直到 v0.28，
对 `/` 的每种方法都会得到该页面。

**大文件上传可以分块进行**（自 v0.29.0 起）。一次 `PUT` 会整体保存在内存中，
最大 1 GiB，因此无法流式传输的客户端，例如浏览器或
[文件资源管理器]({{< relref "/administration/_index.md" >}})，会先发送一个空的
`PUT`，然后发送每个分块，其 `Content-Range` 恰好从上一块结束的位置开始，
最后用 `MOVE` 移到真正的名称。起点超过末尾的范围
仍然返回 `416`：它会留下没有人发送过的字节。
