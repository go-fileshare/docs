---
title: "TLS 与来自 ACME 的证书"
weight: 10
description: "使用来自文件或 ACME CA 的证书，通过 TLS 提供 WebDAV、S3 和 NFS，以及为什么拒绝明文下带密码的 WebDAV。"
tags: [安全, tls, acme]
---

自 v0.9.0 起，**WebDAV 和 S3 通过 HTTPS 提供，NFS 通过
RPC-with-TLS 提供**（RFC 9289），方法是在它们的 `serve` 块上设置 `tls = true`，
并用一个 `tls` 块说明证书的来源。

## 来自文件 {#from-files}

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

文件**在发生变化时会重新加载**，因此其他
工具写入的续期证书无需重启即可生效。证书由
[go-authn/servercert](https://github.com/go-authn/servercert) 提供。

## 来自 ACME CA {#from-an-acme-ca}

默认为 Let's Encrypt；也可以是通过**外部账户
绑定**认识你的 CA，例如通过 HARICA 提供的 GÉANT TCS：

```hcl
tls {
  acme {
    directory_url     = "https://acme-v02.harica.gr/acme/<uuid>/directory"   # cm.harica.gr 显示的 Server URL
    domains           = ["files.example.org"]
    cache_dir         = "/var/lib/fileshare/acme"
    eab_key_id        = "…"
    eab_hmac_key_file = "/etc/fileshare/tls/eab.key"   # 机密：放在文件中，绝不放进配置
  }
}
```

| 字段 | |
|---|---|
| `directory_url` | CA 的 ACME 目录；留空表示 Let's Encrypt |
| `domains` | 要申请的名称（必填） |
| `cache_dir` | 保存账户和证书的位置（必填） |
| `email` | 账户的联系方式，可选 |
| `eab_key_id`、`eab_hmac_key_file` | 外部账户绑定；HMAC 密钥是机密，因此从文件中读取，格式为 CA 发放时的 base64url（标准 base64 也可读取）；其他格式会被拒绝 |
| `http_challenge` | 应答 http-01 的位置，例如 `"0.0.0.0:80"` |

### ACME 能否工作，取决于 CA 如何检查名称 {#whether-acme-can-work-is-decided-by-how-the-ca-checks-the-name}

| 质询 | CA 连接到 | 因此 |
|---|---|---|
| tls-alpn-01 (RFC 8737) | 端口 **443** | 必须在 443 上提供某个 TLS 协议 |
| http-01 (RFC 8555 §8.3) | 端口 **80** | `http_challenge = "0.0.0.0:80"` 来应答它，或使用端口 80 所转发到的地址 |
| 无 | 无 | 域名已**预先验证**的 CA 账户不需要任何质询：对 GÉANT TCS（OV 证书）而言是 HARICA Enterprise **Admin** 账户，因此**外部无法访问**的服务器仍可获得其证书 |

最后一行对校园网内部的文件服务器最为重要：
如果使用的 GÉANT TCS 账户的域名已由机构在 HARICA 验证，
则完全不必向互联网开放任何端口。

#### HARICA（GÉANT TCS）：使用哪种账户，以及 CAA {#harica-gant-tcs-which-account-and-caa}

HARICA 的 ACME 账户有两种，只有其中一种可以跳过
质询：

- **Enterprise Admin** 账户为机构已验证的域名签发 **OV** 证书：
  **无需质询**，这是用于外部无法访问的服务器的账户；
- **Enterprise User** 账户签发 **DV** 证书，而 DV 意味着
  CA 在每次订单中都要检查名称，如上所述使用 tls-alpn-01 或 http-01。

无论哪种情况，域名的 **CAA** 记录（如果有）都必须允许
`harica.gr`，否则订单会被拒绝。

#### 何时申请证书 {#when-the-certificate-is-asked-for}

**在启动时**（自 v0.27.0 起）：监听器就绪后，每个已配置域名的证书
都会被获取，或从缓存目录读回。失败
不会使服务器停止：它继续提供服务，并在 1 分钟后重试，
间隔加倍直至 1 小时，而第一个指明该主机的 TLS 连接无论如何都会再次申请。
证书在到期前 30 天续期，如果其有效期更短，则在
有效期的最后三分之一时续期。

**不发送 SNI 的客户端**，即通过 IP 地址连接的客户端，会得到
**第一个**域名的证书（自 v0.27.0 起）；此前握手会失败。
客户端仍会对照其所拨打的地址检查该证书，因此它
对被告知信任该名称的客户端有帮助，而不会欺骗任何人。

{{< callout type="info" >}}
**ALPN**

使用 ACME 时，服务器的 ALPN 列表包含 `acme-tls/1`，而 Go 服务器
会拒绝其列表与自身没有任何交集的客户端。因此对于 WebDAV 和 S3，会**追加** `http/1.1`
（每个浏览器和 WebDAV 客户端都提供它），
对于 NFS 追加 `sunrpc`，即 RFC 9289 §5.2 用于标识 RPC-with-TLS 的值。不使用 `h2`：
这些监听器不提供 HTTP/2，提供它就是撒谎。
`acme-tls/1` 保留在每个监听器上，因此 CA 无论从哪里访问，
tls-alpn-01 都能得到应答。
{{< /callout >}}

## WebDAV 不以明文提供 {#webdav-is-not-served-in-the-clear}

{{< callout type="error" >}}
**v0.9.0 中的破坏性变更：可访问地址上带密码的 WebDAV 会被拒绝**

HTTP Basic 在每个请求中就是经过 base64 编码的密码，而持有者令牌
与密码等效。因此，在其他机器可以访问的地址上提供**没有 TLS** 的
WebDAV，而又有人需要身份验证的配置会被
**拒绝，而不是仅仅警告**：警告一闪而过，而泄露的密码
无法收回：

```
webdav on 0.0.0.0:8080 would carry passwords in the clear: serve it with
tls = true, or -- when TLS is terminated in front of it, by a proxy -- say
plaintext = true
```
{{< /callout >}}

在 v0.9.0 之前以
`serve "webdav" { addr = "0.0.0.0:8080" }` 启动的配置，在此之后无法启动。有两种
解决办法：

```hcl
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true            # 配合 tls 块
}
```

或者，当反向代理在前端终结 TLS 并以明文转发时：

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true       # 有意在其前端终结 TLS
}
```

**回环**地址（`127.0.0.1`、`::1`、`localhost`），以及**没有任何人需要
身份验证**的服务器（没有 `user`、没有 `users`、没有 `oidc` 块）
不受影响。没有 `addr` 的 `serve` 块落在回环地址上，同样不受影响。

`plaintext` 仅适用于 WebDAV，即会发送密码的协议，
而 `plaintext` 与 `tls` 同时出现会被拒绝：二者取其一。

## `check` 会说明什么 {#what-check-says}

[`fileshare check`]({{< relref "/configuration/check.md" >}})会按协议打印哪些是
加密的，哪些是**有意**明文的：

```
webdav: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem
nfs: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem; clients present one signed by /etc/fileshare/tls/clients.pem (the machine, not the person)
```

对于代理之后的 WebDAV，则为

```
webdav: in the clear on 10.0.0.5:8080, on purpose (plaintext = true): TLS must be terminated in front of it
```

## 按协议 {#per-protocol}

| 协议 | `tls = true` | |
|---|---|---|
| WebDAV | 是 | HTTPS |
| S3 | 是 | HTTPS |
| NFS | 是 | RFC 9289；见下文 |
| SMB | **拒绝** | SMB 3 使用自己的密钥加密 |
| SFTP | **拒绝** | SFTP 就是 SSH |

## 基于 TLS 的 NFS 证明的是机器，而不是个人 {#nfs-over-tls-proves-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

`client_ca_file` 会要求客户端出示由该颁发机构签发的证书，由此得知
**是哪台主机在挂载**，而 RFC 9289 对用户身份验证保持原样：
其中的 uid 仍然是 `AUTH_SYS` 所声明的那个。因此，指明了谁可以使用的共享，
在没有 `kerberos` 块时**通过 NFS 仍然会被拒绝**，在没有
[基于证书的身份]({{< relref "/security/nfs-certificates.md" >}})时也一样，后者是对
同一证书的另一种用法。

TLS 是**可选提供，而非强制**的：从不请求它的客户端仍然可以使用
开放的共享。没有 `tls = true` 的 `client_ca_file` 会被拒绝，因为它
什么都不会检查，而在 NFS 以外的任何协议上使用 `client_ca_file`
同样会被拒绝。

## 还有哪些会被拒绝 {#what-else-is-refused}

- 没有 `tls` 块的 `tls = true`：没有任何内容说明证书来自哪里。
- 没有任何 `serve` 块使用的 `tls` 块：没有任何东西会用到它。
- 不是可监听地址的 `http_challenge`。
