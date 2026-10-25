---
title: "配置"
linkTitle: "配置"
weight: 10
description: "HCL 配置的组织方式，user 块可以携带什么，以及启动时会拒绝什么。"
tags: [配置]
---

一个文件，或一个由若干小文件组成的目录。`--config /etc/fileshare.d` 会将
它们合并，因此一个文件中的用户和另一个文件中的共享可以关联在一起。

```hcl
name = "ATTIC"

user "alice" { password_file = "/etc/fileshare/alice.pw" }
user "bob"   { password_file = "/etc/fileshare/bob.pw" }

group "family" { members = ["alice", "bob"] }

share "photos" {
  image   = "/srv/photos.img"
  allow   = ["@family"]        # 两个列表中都可以写组或个人
  writers = ["alice"]          # bob 只有只读权限
}

share "scratch" {
  image = "/srv/scratch.img"   # 任何通过身份验证的人，可读写
}

tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true                  # HTTP Basic 即密码本身：绝不能明文传输
}
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

{{< callout type="error" >}}
**自 v0.9.0 起，带密码的 WebDAV 不再以明文提供服务**

`serve "webdav" { addr = "0.0.0.0:8080" }`，即此前本页展示的示例，
现在只要有人需要身份验证就会**在启动时被拒绝**：
每个请求中的 HTTP Basic 就是密码本身。请如上所示配合 `tls` 块使用
`tls = true`，或者在代理于前端终结 TLS 时使用
`plaintext = true`。参见 [TLS]({{< relref "/security/tls.md" >}})。
{{< /callout >}}

共享也可以是[设备]({{< relref "/configuration/shares.md#a-device-not-only-an-image" >}})或
[主机上的目录]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})，而不一定是
映像。

## `user` 块可以携带什么 {#what-a-user-block-can-carry}

上面的每个示例都使用 `password_file`，这是最常见的情形，
也曾有一段时间是本程序唯一读取的字段。但它并不是整个块，
而其中的区别决定了**一个人可以通过哪些协议获得服务**，
因此值得集中说明，而不是等到挂载时才发现。

| 字段 | 含义 | 作用 |
|---|---|---|
| `password` | 写在文件中的密码 | 所有可用密码应答的协议 |
| `password_file` | 同上，但放在单独的文件中 | 同上，且配置中不含机密 |
| `nt_hash` | `MD4(UTF16LE(password))`，32 个十六进制字符 | **SMB**，用于本站点并不持有其密码的人员 |
| `authorized_keys` | 内联的 `authorized_keys` 行 | **SFTP** |
| `authorized_keys_file` | 同上，来自文件 | **SFTP** |
| `totp_secret` | base32 编码的一次性验证码密钥 | 目前没有作用：它会被读取并检查，但这里没有任何协议要求第二因素 |

⛔ 同时给出 `password` 和 `password_file` 会在启动时被拒绝，并指明
对应的人员：对“其密码是什么”给出两个答案，就产生了哪一个生效的问题，
而配置不应需要读两遍才能弄清楚。同时给出
`authorized_keys` 和 `authorized_keys_file` 也是如此。

⛔ 不含 `password`、`password_file`、`authorized_keys`
或 `authorized_keys_file` 中任何一个的 `user` 块同样会被拒绝，提示 *has no way to authenticate*，
除非设置了 `trusted_user_ca_file`。`nt_hash` 和 `totp_secret` 在此
不算数。

{{< callout type="default" >}}
**内联用户可以通过 SMB 提供服务**

直到最近，这个块都只读取密码字段，因此要为这里写明的人员
提供 SMB 服务，就意味着在此文件中以明文提供其密码，
或者把这些人员移到数据库中。有了 `nt_hash`，就不必如此：
存储 Samba 所存储内容的站点可以改写这个值。该块仍然
需要旁边有上述字段之一（例如密钥），或者
`trusted_user_ca_file`：仅有 `nt_hash` 会在启动时被拒绝。参见
[来源能够证明什么]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}})，
它是上一层的同一张表。
{{< /callout >}}

在任何可以写人员的地方，组都写作 `@name`。

## 在启动时而不是之后被拒绝的内容 {#what-is-refused-at-startup-rather-than-later}

{{< callout type="warning" >}}
**不属于任何人的名称**

否则 `allow = ["alise"]` 会把 Alice 挡在 Alice 自己的共享之外，
却仍然正常启动。没有任何 `user` 块、也没有任何 [`users` 目录服务]({{< relref "/configuration/identity.md" >}})
认识的名称会被拒绝，没有任何一方拥有的 `@group` 同样如此。
{{< /callout >}}

{{< callout type="warning" >}}
**指明了谁可以使用、却要通过 NFS 提供的共享**

一个声明 *photos 属于 alice* 的配置，和一个把
photos 交给任何连接者的协议，二者不可能同时得到遵守。这样的共享不会
通过 NFS 导出，除非 `kerberos` 块或 `identity = "certificate"`
让 NFS 能够区分不同的人；而没有任何内容可供承载的 `nfs` serve 块
会被拒绝。参见 [NFS]({{< relref "/protocols/nfs.md" >}})。
{{< /callout >}}

{{< callout type="warning" >}}
**无处安放其凭证的提供方**

除非提供了 WebDAV（唯一携带 `Authorization` 标头的协议），
或者提供了 SFTP 并启用了提供方的证书（`ssh_ca_file` 或
`opkssh_client_id`，参见 [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})），
否则 [`oidc`]({{< relref "/protocols/webdav.md" >}}) 块会被拒绝。
SMB 和 NFS 对这两者都无处安放。
{{< /callout >}}

{{< callout type="warning" >}}
**明文密码**

WebDAV 位于其他机器可以访问的地址上，没有 `tls = true` 或
`plaintext = true`，而又有人需要身份验证。参见
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})。
{{< /callout >}}

{{< callout type="warning" >}}
**无法安全提供服务的块**

没有 `state_file` 的 `admin` 块，或者在 TCP 上监听而没有双向
TLS 的 `admin` 块；没有任何 `serve` 块使用的 `tls` 块；短于一秒的
`reload`；通过明文 HTTP 获取的撤销列表。每一项
都在相应的位置有说明：[管理]({{< relref "/administration/_index.md" >}})、
[TLS]({{< relref "/security/tls.md" >}})、[重新加载]({{< relref "/administration/reload.md" >}})、
[撤销]({{< relref "/security/_index.md" >}})。
{{< /callout >}}

{{< callout type="warning" >}}
**此二进制文件构建时未包含的协议**

系统会告知*这一点*，而不是“没有这样的协议”，这就是
拼写错误与[构建标签]({{< relref "/operations/build-tags.md" >}})之间的区别。
{{< /callout >}}

没有 `addr` 的 `serve` 块在回环地址 `127.0.0.1` 上监听，使用
无需特权的端口：SMB 为 4445，WebDAV 为 8080，SFTP 为 2222，NFS 为 2049，
S3 为 9000。

## 共享上的 `protocols` {#protocols-on-a-share}

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

它指明由哪些协议承载该共享。它本身就有用：声明为仅限 SMB 的共享，
WebDAV 进程永远不会得知它；在某些
配置中，[`--isolate`]({{< relref "/operations/isolation.md" >}}) 也要求使用它。最终不承载任何内容的
`serve` 块同样会被拒绝，并且发生在打开任何映像之前，
同时指明被它排除的共享及原因，除非有 `admin` 块
稍后为它提供共享。
