---
title: "SFTP：密钥或证书"
linkTitle: "SFTP：密钥或证书"
weight: 30
description: "使用公钥或 SSH 证书的 SFTP：本地和提供方的颁发机构、OpenPubkey、域授权、源地址以及撤销。"
tags: [协议, sftp, ssh]
---

每个客户端都已经具备它：`sftp` 随 OpenSSH 一起提供，Finder 和 GNOME
可以挂载它，编辑器也支持它。它也是这里唯一形态并不契合的协议：
一个人登录后进入的是**一个**文件系统，而不是一个共享列表。

因此，共享成为为刚刚通过身份验证的人构建的目录树的顶层目录：

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

alice 无权使用的共享**不是 alice 能看到的目录**，而 alice
只能读取的共享，会在*目录树中*就拒绝 alice 的写入，
而不是交给本来会允许写入的驱动。

{{< callout type="info" >}}
**跨两个共享的重命名会被拒绝**

那将是在两个映像之间进行一次复制和一次删除，而这在任何地方
都不是重命名所承诺的行为。
{{< /callout >}}

## 配置 {#configuration}

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # 可选

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # 这里什么都没有：carol 的证书就是其凭据
```

配置了 `trusted_user_ca_file` 后，由该颁发机构签发、并在其主体中包含
该用户的证书就已足够，因此一个人的访问权限是在**别处签发并在别处到期**的，
有人加入或离开时，这里无需编辑任何文件。

签名、有效期和主体由
`x/crypto/ssh` 的 `CertChecker` 检查；并已对照 **OpenSSH 自己的客户端**验证，
当其证书被移开后，该客户端同样会拒绝这把密钥。

{{< callout type="warning" >}}
**没有 `host_key_file` 时，每次启动都会生成新的身份**

服务器会说明这一点，而所有以前见过它的客户端都会就
密钥变更发出警告，这正是客户端在履行其职责。
{{< /callout >}}

## 针对本主机的证书：域授权 {#certificates-meant-for-this-host-the-domain-grant}

自 v0.22.0 起。[EuroHPC 联合平台的 SSH CA](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-overview/)
（GÉANT / MyAccessID）签发联盟中每个站点
都信任的证书。每个证书都在
`ssh-domain-grant@core.aai.geant.org` 扩展中说明它适用于哪些主机。sshd 会忽略该扩展，
fileshare 在 v0.22.0 之前也是如此：为他人
机器签发的证书在这里同样有效。设置 `ssh_domains` 后，授权会由
[go-authn/sshcert](https://github.com/go-authn/sshcert) 按规范读取、解析
和匹配。授权中没有包含本主机任何名称的证书
会被拒绝。

```hcl
trusted_user_ca_file = "/etc/fileshare/efp-ssh-ca.pub"
ssh_domains          = ["files.example.org", "sftp.example.org"]
# ssh_accept_ungranted = true   # 仅当受信任的 CA 从不写入授权时
```

| 键 | |
|---|---|
| `ssh_domains` | 本主机的名称，按授权指明它们的方式书写。只要证书的授权中有一个模式匹配其中一个名称，该证书就会被接受。`*` 表示**单个标签内的一个或多个字符**：`*.example.org` 授权 `files.example.org`，但不授权 `a.files.example.org`，而 `files-*.example.org` 不授权 `files-.example.org`。比较时忽略大小写。这些是主机名，而不是模式，每个只列出一次，并且至少必须信任一个颁发机构（`trusted_user_ca_file`，或 `oidc` 块的 `ssh_ca_file`）。 |
| `ssh_accept_ungranted` | 放行**没有**授权的证书。默认关闭；没有 `ssh_domains` 时，因其毫无意义而被拒绝。 |

| 证书 | `ssh_domains` | `+ ssh_accept_ungranted` | 无 `ssh_domains` |
|---|---|---|---|
| 授权了 `ssh_domains` 之一 | 放行 | 放行 | 放行 |
| 仅授权了其他主机，或 `[]` | 拒绝 | 拒绝 | 放行 |
| 无授权 | 拒绝 | 放行，并记入日志 | 放行 |
| 无法解析的授权（`null`、`[1]`、不紧凑……） | 拒绝 | **拒绝** | 放行 |

没有 `ssh_domains` 时，不会读取任何内容，也不会有任何变化。授权是在
**颁发机构**签发的证书上读取的：即 `trusted_user_ca_file` 的证书，以及
提供方的证书（`ssh_ca_file`）。OpenPubkey 证书由用户自己的
密钥签名，因此其中的授权只是用户对自己的说法。它的受众
是 `opkssh_client_id`。普通密钥不携带授权，也不在此规则范围内。

每次拒绝都会连同证书的序列号、密钥 ID 和原因一起记入日志：

```
sftp: alice: certificate 42 ("alice@efp") refused: its domain grant [login.example.eu] names none of this host's names [files.example.org sftp.example.org]
sftp: alice: certificate 43 ("staff") refused: it has no domain grant (ssh-domain-grant@core.aai.geant.org), and ssh_domains requires one
```

每项决定也会计入 `fileshare_sftp_domain_grant_total{result}`，
结果为 `granted`、`accepted_ungranted`、`refused_not_granted`、
`refused_absent` 或 `refused_malformed`。客户端在证明自己持有密钥之前就会出示证书，
因此这里统计的是尝试次数，而不是人数。

{{< callout type="error" >}}
**为什么以拒绝为默认**

设想一台信任第二个颁发机构的主机：它自己用于员工的 CA、测试
CA、附加到同一文件中的 EFP 预发布 CA，或没有授权的 go-authn/bridge
客户端。该颁发机构的证书不携带授权。如果缺少
授权意味着“没有限制”，那么这个过滤器在该主机上就什么都过滤不掉。

在这里，没有授权的证书会被拒绝，除非 `ssh_accept_ungranted`
另有说明。存在但无法解析的授权无论内容如何都会被拒绝：写入授权的
颁发机构本意就是限制该证书。go-authn/sshcert 演示了
[多 CA 场景](https://github.com/go-authn/sshcert#why-fail-closed-the-multi-ca-scenario)。
{{< /callout >}}

### 使用 EFP 的 CA {#with-efps-ca}

`trusted_user_ca_file` 即 OpenSSH 的 `TrustedUserCAKeys`。按照 EFP 的
[信任页面](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-trust/)所述，将 EFP
发布于 <https://sshca.my-eurohpc.eu/config> 的密钥放入其中。
该 URL 返回 `{"PublicKey": "ssh-ed25519 ..."}`；文件中单独一行保存
`PublicKey` 的值：

```sh
curl -s https://sshca.my-eurohpc.eu/config | jq -r '.PublicKey' > /etc/fileshare/efp-ssh-ca.pub
```

然后将 `ssh_domains` 设置为你的托管实体的授权所使用的名称。EFP
证书的唯一主体是 MyAccessID 标识符
（`<id>@myaccessid.org`），因此这就是它登录时使用的用户名。同名且
没有自己凭据的 `user` 块会接收它：

```hcl
user "u1234@myaccessid.org" {}
```

### 使用 go-authn/bridge {#with-go-authnbridge}

设置了 `ssh_certificates = true` 和
`ssh_domain_grants = ["files.example.org"]` 的
[go-authn/bridge](https://github.com/go-authn/bridge) 客户端（v0.19.0 或更高版本）会签发针对
本主机授权的证书。设置 `ssh_principal_claim = "voperson_id"` 后，其唯一主体是
此人的 `voperson_id`，与 EFP 配置一致，这就是要在此处
编写的 `user` 块。信任 bridge 的 CA，与本服务器信任其他任何 CA 的方式相同：

- 作为 `trusted_user_ca_file`，用于本地账户。bridge 在 `/ssh/config` 以 EFP
  的格式提供其密钥，因此上面的 `curl | jq` 命令配合 bridge 的 URL 同样有效；
- 或作为 `oidc` 块的 `ssh_ca_file`，用于
  [由提供方担保的人员](#people-the-identity-provider-vouches-for)。

bridge 的其他客户端不写入授权。如果它们的证书必须在此
继续有效，这正是 `ssh_accept_ungranted` 的用途。但它会放行每个受信任颁发机构的**所有**
无授权证书，所以最好还是为这些
客户端提供授权。

### 绑定到某个地址的证书 {#a-certificate-pinned-to-an-address}

bridge 的 `ssh_source_address` 会写入 `source-address` 关键选项，与
`ssh-keygen -O source-address=...` 的效果相同。携带该选项的证书
**只接受来自这些地址的登录**，无论是否设置了 `ssh_domains`。来自
证书所允许地址的登录会被放行，前提是在设置了 `ssh_domains` 时，
其授权也包含本主机。来自任何其他地址的登录会
被拒绝。

| fileshare | `trusted_user_ca_file` | `oidc` 的 `ssh_ca_file`，以及 OpenPubkey |
|---|---|---|
| v0.22.0 之前 | 强制执行 | **所有地址均被拒绝** |
| v0.22.0，无 `ssh_domains` | 强制执行 | **所有地址均被拒绝** |
| v0.22.0，有 `ssh_domains` | **所有地址均被拒绝** | **所有地址均被拒绝** |
| v0.22.1，无论有无 `ssh_domains` | 强制执行 | 强制执行 |

“所有地址均被拒绝”是以拒绝为默认的：证书的限制从未
放宽，只是登录丢失了。在 v0.22.1 之前，go-filesystems/sftp 的
sshd 会拒绝它交给 fileshare 决定的证书上的任何关键选项；
这涵盖提供方的证书、OpenPubkey 证书，以及在设置 `ssh_domains` 时的
所有证书。v0.22.1 构建于 go-filesystems/sftp v0.5.1 之上，其 sshd
在那里接受 `source-address`，并在每条证书路径上依据连接来源的
地址（IPv4 或 IPv6）强制执行。

其他任何关键选项（`force-command`、`verify-required` 等）都会被拒绝，
因为本服务器不会对其采取任何动作。

## 由身份提供方担保的人员 {#people-the-identity-provider-vouches-for}

自 v0.11.0 起，`oidc` 块也可用于 SFTP，但使用的不是令牌（SSH
没有地方放置令牌），而是**证书**：

```hcl
oidc {
  issuer           = "https://login.example.org"
  audience         = "fileshare"
  ssh_ca_file      = "/etc/fileshare/bridge-ca.pub"   # go-authn/bridge 的 ssh_ca
  opkssh_client_id = "opkssh"                         # OpenPubkey 登录
  opkssh_max_age   = "24h"                            # 12h、24h、48h、1week
}
```

- **由提供方的 SSH CA 签发的证书**：`authn-bridge ssh-cert` 会在通过联盟登录之后
  写入一份。其主体是此人，其
  `groups@go-authn.org` 扩展是此人所属的组，因此
  [`oidc:groups:` 规则]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})
  适用。它**不是** `trusted_user_ca_file`，后者的证书针对
  本地账户：本地颁发机构的证书若声称拥有提供方的组，
  则会被视为它所指明的本地账户。
- **OpenPubkey 证书**，由 `opkssh login` 写入：由用户自己的
  密钥签名，并带有承诺该密钥的 ID 令牌。检查方式与
  `opkssh verify` 相同：先由
  [openpubkey](https://github.com/openpubkey/openpubkey) 验证器检查（对照提供方已发布的密钥检查
  其签名、nonce 承诺、
  客户端 ID 和有效期），再将证书的密钥与令牌中的密钥比对；并且
  SSH 用户名必须是令牌的用户名，因为这里没有 `auth_id` 文件
  可以把二者对应起来：`sftp alice@univ-example.fr@files.example.org`。
  `opkssh_client_id` 应当是专属的客户端 ID，绝不能是其他
  应用的：ID 令牌会传送到此人登录的每个服务器。

要查看证书所携带的组：

```sh
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub     # Extensions 部分
```

并且服务器会在每次联盟登录时说明
`sftp: alice@univ-a.fr, vouched for by the provider, in groups [...]`。

{{< callout type="error" >}}
**被担保不等于被准入**

除非有规则指明某人、`trust_all` 表示提供方就是目录服务，或者
`local_names = true` 表示提供方的名称就是本服务器的名称且本地
账户拥有该名称，否则提供方所担保的人在这里仍然是陌生人，这与令牌在 WebDAV 上
所通过的测试相同。**没有主体**的提供方证书，
按格式自身的定义对*任何人*都有效，因此会被拒绝。
{{< /callout >}}

`-tags noopenpubkey` 会排除 OpenPubkey 验证器（约 2 MB）；
此类构建中带有 `opkssh_client_id` 的配置会被拒绝。

## 收回证书 {#taking-a-certificate-back}

{{< callout type="error" >}}
**证书在登录时检查**

在提供方处撤销某人，本身并不会撤销已经签发的
证书：该证书在到期之前仍可打开 SFTP，而已经打开的 SFTP
会话在那之前也保持打开，因为这些人不在目录服务中，所以
[重新加载]({{< relref "/administration/reload.md" >}})与这些人员无关。窗口期就是
证书的有效期：bridge 的 `ssh_ca { validity }`（默认 12h，最长
168h，且不超过 IdP 会话的结束时间）以及这里的 `opkssh_max_age`。请在客户端重新登录
允许的范围内尽可能缩短。自 v0.17.0 起，已经打开的 opkssh
会话也会在 `opkssh_max_age` 时结束，无论
证书（由用户自己签名）怎么说。
{{< /callout >}}

有两种机制可以关闭该窗口期：

- 提供方的 **[KRL]({{< relref "/security/revocation-lists.md" >}})**（`ssh_krl_url`）：
  被撤销的证书在登录时被拒绝，而由它打开的会话的每个操作
  都会再次查询该列表，包括已打开的文件；
- **[共享信号]({{< relref "/security/shared-signals.md" >}})**（`ssf`）：在 CAEP `session-revoked` 之前，提供方为此人
  签发的一切内容都会被拒绝，
  这是处理不在任何列表中的 OpenPubkey 证书的唯一途径。

两者都以拒绝为默认。

## 没有密码 {#no-password}

提示输入密码的客户端正在做密钥本来要避免的事情。

## 传输大小与打开的文件 {#transfer-size-and-open-files}

fileshare 会应答 OpenSSH 的 `limits@openssh.com`（自 v0.25.0 起），因此
OpenSSH 的 `sftp` 每个请求最多读写 255 KiB，而不是
32 KiB：使用 OpenSSH 10.3 实测约提升 25%。一个会话最多可同时持有
1024 个打开的文件和目录；在目录共享上，每一个都是主机的
一个描述符，这个上限可防止单个用户耗尽
所有描述符。
