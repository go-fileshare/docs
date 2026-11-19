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

## Kerberos：用票据代替密码 {#kerberos}

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"   # nfs/fs.example.org 和 cifs/fs.example.org
}

serve "smb" {
  addr = "0.0.0.0:445"
  ntlm = false          # 仅 Kerberos：从不校验任何密码
}
```

让 NFS 能够区分人员的 `kerberos` 块，同样让 SMB 接受票据。**一个 keytab** 同时保存两个服务，
每个协议只应答自己的服务：SMB 只读取该块所在领域（realm）的 `cifs/<主机>` 主体，所以为
`nfs/` 签发的票据在这里会被拒绝。没有 `cifs/` 主体的 keytab 会让 SMB 保持原样，只用 NTLM，
服务器会在启动时说明这一点。

主体先经过与 NFS 相同的规则（另一个领域的 `alice` 什么人也不是），然后必须是**这份配置所认识的人**：
票据只证明 KDC 认识这个人，并不证明这台服务器认识。共享的 `allow` 和 `writers` 列表与使用密码时
一样生效。会话随票据一同结束，Windows 客户端会自行重新认证。

自 fileshare v0.50.0 起，**keytab** 必须是普通文件，属于 root 或运行 fileshare 的账户，
且不得被其他人读写，也不得被其所属组写入：`root:fileshare` `0640` 和
`fileshare:fileshare` `0600` 会被接受，`krb5:fileshare` `0640` 会被拒绝。这同时适用于
SMB 和 NFS。客户端的名称和领域取自**票据**，绝不取自客户端一并发送的认证器
（jcmturner/gokrb5#577，RFC 4120 第 3.2.3 节）；NFS 需要 go-authn/krb5 v0.3.4 或更高版本，
v0.50.0 已包含。

没有 `kerberos` 块时，`ntlm = false` 在加载时即被拒绝；keytab 中没有该领域的 `cifs/` 密钥时，服务器
拒绝启动：没人能登录的服务器不算服务器。有了该块，目录里没有 NT 哈希的人也能使用 SMB，
[`check`]({{< relref "/configuration/check.md" >}}) 会如实显示。

**未加入域**的 Windows 机器用本领域的票据访问：执行一次 `ksetup /addkdc` 和
`ksetup /addhosttorealmmap`，在登录会话中放入票据授予票据，然后执行
`net use \\fs.example.org\photos`——要用名称，因为使用地址时 Windows 会回退到 NTLM。
用 Windows 11 25H2（未加入域）实测：客户端提供 MS-KRB5、Kerberos、NEGOEX 和 NTLMSSP，
发送 Kerberos AP-REQ，在要求签名的情况下挂载、列目录、读取和写入，并且不发送任何 NTLM 消息。

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

## 并发处理的请求 {#requests-in-flight}

自 go-filesystems/smb v0.8.1 起，一个连接最多同时处理 32 个请求，整个服务器跨所有连接最多 128 个。回复按 `MessageId` 匹配。仅含一个普通文件 `READ` 的帧可与其他请求并行执行；复合链（整体按顺序应答）、异步请求和 `WRITE` 单独执行。一个帧开始后须在 30 秒内到达。该机制是共享模块 go-filesystems/inflight，SFTP 和 NFS 也使用它。
