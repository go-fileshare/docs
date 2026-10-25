---
title: "撤销 SSH 证书：KRL"
weight: 20
description: "使用带签名、带日期的 OpenSSH KRL 撤销身份提供方的 SSH 证书，并在登录时以及每次 SFTP 操作时检查。"
tags: [安全, 撤销, sftp, ssh]
---

身份提供方的 SSH CA 签发的证书（参见
[针对提供方所担保人员的 SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})）
在登录时检查。在 go-authn/bridge 中停用某人会撤销其
令牌并删除其应用密码，这一点由
[目录服务重新加载]({{< relref "/administration/reload.md" >}})在此处应用；但没有
列表的话，已经签发的证书在到期之前仍可打开 SFTP，
而已经打开的 SFTP 会话也会保持打开。

自 v0.12.0 起，列表可以弥补这一点：

```hcl
oidc {
  issuer          = "https://login.example.org"
  audience        = "fileshare"
  ssh_ca_file     = "/etc/fileshare/bridge-ca.pub"
  ssh_krl_url        = "https://bridge.example.org/ssh/krl"   # 或 ssh_krl_file
  ssh_krl_state_file = "/var/lib/fileshare/ssh.krl.state"    # 跨重启保持先后顺序
  ssh_krl_max_age    = "1h"                                  # 默认值；ssh_krl_refresh = "1m"
}
```

go-authn/bridge 会把它已撤销的 SSH 证书（被停用的个人、
被停用的 IdP）发布为 OpenSSH **KRL**（`PROTOCOL.krl`，即
`sshd` 的 `RevokedKeys` 读取的列表），而 fileshare 保留其副本，并用
[go-authn/krl](https://github.com/go-authn/krl) 读取。

## 在登录时拒绝，在会话中途也拒绝 {#refused-at-login-and-in-the-middle-of-a-session}

被撤销的证书在登录时被拒绝，**它已经打开的会话
也不再获得服务**。SSH 只在握手时检查一次证书；因此联盟 SFTP 会话的每次
操作都会再次查询 KRL，**包括已打开的文件**。本服务器的列表副本发生变化之后的第一次操作就会被拒绝。

## 字段 {#fields}

| 字段 | 默认值 | |
|---|---|---|
| `ssh_krl_url` | | 获取列表的位置，**仅限 HTTPS** |
| `ssh_krl_file` | | 或文件，使用绝对路径，二者取其一，不可同时给出 |
| `ssh_krl_ca_file` | 系统的 | 固定用于检查列表 HTTPS 服务器的颁发机构 |
| `ssh_krl_refresh` | `1m` | 获取的频率，至少一秒 |
| `ssh_krl_max_age` | `1h` | 最近一份有效副本可以多旧，超过后它所管辖的证书会被拒绝 |
| `ssh_krl_state_file` | | 保存最近一份通过验证的列表的位置，以便重启后记得先后顺序 |

短于 `refresh` 的 `max_age` 会被拒绝，因为每一份副本在
下一次获取之前都已过期。没有 `ssh_ca_file` 的 KRL 同样会被拒绝：它撤销
提供方的证书，而没有该 CA 就没有这样的证书。

## 它以拒绝为默认 {#it-fails-closed}

{{< callout type="error" >}}
**在无法获取 KRL 期间，提供方的证书会被拒绝**

在无法获取列表，或最近一份有效副本比
`ssh_krl_max_age` 更旧时，提供方的 CA 签发的**每一份**证书都会
被拒绝：在列表无法访问时放行一切的撤销检查，会被能够阻断该列表的
攻击者击败。因此列表
必须与它所管辖的登录一样可靠地提供服务。
`fileshare_revocation_list_age_seconds{list="ssh_krl"}` 是应当在到达 `max_age`
之前设置告警的指标；参见
[健康检查与指标]({{< relref "/administration/health.md#what-to-alert-on" >}})。
{{< /callout >}}

## 带签名、带日期，来自 `ssh_krl_url` {#signed-and-dated-from-ssh_krl_url}

自 v0.15.0 起，**来自 `ssh_krl_url` 的 KRL 必须带有签名，并说明何时
过期**，如 [go-authn/revocation](https://github.com/go-authn/revocation)
所述，go-authn/bridge 自 v0.10.0 起提供：

- **签名：** `<url>.sig` 是提供方 SSH CA（`ssh_ca_file`，因此
  `ssh_krl_url` 要求提供它）在命名空间 `krl@go-authn.github.io` 中的分离式 SSHSIG 签名。
  获取时对列表的 ETag 使用 `If-Match`，如果列表在此期间被重新签发，
  则立即再次获取。
- **过期时间：** `expires@go-authn.github.io` 扩展。过了该时间，列表
  就不再是最新的，无论多近的 `304` 确认过它。

HTTPS（`ssh_krl_ca_file` 固定其颁发机构）验证的是应答的服务器，
而不是列表：镜像、缓存或被攻陷的 Web 服务器都可能
提供空列表，而仅凭 KRL 本身无法看出它已过期。未通过验证的列表
会被拒绝，并保留最近一份有效副本。通过 `ssh_krl_file` 手工放置的列表，
其可信度等同于保存它的文件系统，不需要
签名或过期时间；若其中带有，则予以遵守。

{{< callout type="warning" >}}
**升级到 v0.15.0**

提供未签名 KRL 或没有过期时间的 `ssh_krl_url`
不再被接受：此时提供方的证书会被拒绝，与任何
无法获得的列表一样。请先运行 go-authn/bridge v0.10.0 或更高版本。
{{< /callout >}}

## 绝不倒退，重启之后也一样 {#never-backwards-across-a-restart-too}

自 v0.16.0 起，比已持有的列表更旧的列表会被拒绝：版本更低
（KRL 的版本、CRL 的编号），或版本相同但签发更早。配合 `ssh_krl_state_file`
（[NFS]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})对应的是 `crl_state_file`），最近一份通过验证的列表会
保存在磁盘上，在启动时再次验证，并决定下一份列表的先后。它本身
在获取到列表之前不被视为最新。没有它，重启
会忘记先后顺序，fileshare 会在启动时说明这一点：较旧的列表，只要仍有签名
且未过期，就会被接受。

这些文件，与 `ssh_ca_file`、各 CA 文件、列表文件，以及（自
v0.17.0 起）`authorized_keys_file`、`dsn_file`、`bind_password_file` 和
`ssf` 的 `ca_file` 一样，**不得位于共享之内**：能在那里写入的人
就能决定谁可以进入。

## 与 fileshare 并存的普通 `sshd` {#a-plain-sshd-next-to-fileshare}

同样的验证无需 fileshare 也能使用：go-authn/revocation 的
`authn-revokd` 获取列表，只保留通过验证的列表，在磁盘上保持它们的先后顺序，
为 `sshd` 写入 `RevokedKeys`，并且在列表失效时，写入一份
撤销 CA 本身的列表，使 `sshd` 同样以拒绝为默认。

## 它不涵盖什么 {#what-it-does-not-cover}

**OpenPubkey（opkssh）证书不在任何 KRL 中**，因为只有
此人自己的密钥签发过它们。它们受 `opkssh_max_age` 限制，并由
[共享信号]({{< relref "/security/shared-signals.md" >}})收回。WebDAV 上的访问令牌同样如此。

同样的机制（获取、保存、以拒绝为默认）也承载着
[基于证书的 NFS 身份]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})的 **X.509 CRL**。
