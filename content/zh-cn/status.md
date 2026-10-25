---
title: "状态"
weight: 60
description: "本文档描述的是哪个 fileshare 版本，每个版本新增了什么，哪些已经用真实客户端验证，以及哪些尚未实现。"
tags: [状态, 发布]
---

本文档描述的是 **fileshare v0.31.0**。

## 每个版本新增了什么 {#what-each-release-added}

| | |
|---|---|
| v0.7.0 | 通过 NFS 提供受限共享，使用 [`kerberos` 块]({{< relref "/protocols/nfs.md#the-refusal-that-used-to-be-permanent" >}}) |
| v0.8.0 | 基于 gRPC 的[管理 API]({{< relref "/administration/_index.md" >}})；[目录共享]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})；[`/healthz`、`/readyz`、`/metrics`]({{< relref "/administration/health.md" >}}) |
| v0.9.0 | 用于 WebDAV、S3 和 NFS 的 [TLS]({{< relref "/security/tls.md" >}})，证书来自文件或 ACME；**拒绝明文下带密码的 WebDAV** |
| v0.10.0 | [重新读取目录服务]({{< relref "/administration/reload.md" >}})：`reload = "5m"`、`SIGHUP`、`ReloadDirectory` |
| v0.11.0 | [针对身份提供方所担保人员的 SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})：其 SSH CA，以及 opkssh |
| v0.12.0 | 撤销：针对提供方 SSH 证书的 [KRL]({{< relref "/security/revocation-lists.md" >}})，针对[基于证书的 NFS 身份]({{< relref "/security/nfs-certificates.md" >}})的 CRL |
| v0.13.0 | [共享信号]({{< relref "/security/shared-signals.md" >}})：CAEP `session-revoked` 会使提供方此前签发的一切在所有协议上失效 |
| v0.14.0 | 安全审查的修复：[协议能够承载的共享名称]({{< relref "/administration/_index.md#what-it-will-and-will-not-touch" >}})、任何共享都不得包含配置或其机密、启动时再次检查 `source_roots`、SSF 的 `max_age` ≥ 1m 且 `retain` ≥ 169h |
| v0.15.0 | [来自 `ssh_krl_url` 的 KRL 必须带有签名和日期]({{< relref "/security/revocation-lists.md#signed-and-dated-from-ssh_krl_url" >}}) |
| v0.16.0 | 撤销列表[绝不倒退]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}})，重启之后也一样，借助 `ssh_krl_state_file` / `crl_state_file`；信任锚和列表文件不得位于共享之内 |
| v0.16.1 | 要求 Go 1.26.6：1.26.4 中有九个可触及的标准库安全公告 |
| v0.16.2 | [NFS CRL 的签名在解析之前检查]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}) |
| v0.16.3 | [不再用未经验证的电子邮件来命名任何人]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}) |
| v0.16.4 | 仅测试 |
| v0.16.5 | 仅测试（[#36](https://github.com/go-fileshare/fileshare/issues/36)：某个测试把传输错误当成了成功） |
| v0.16.6 | NFS 的拒绝信息指出两条出路：`kerberos` 块，或 [`identity = "certificate"`]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.16.7 | go-authn/oidc v0.2.2：RSA 密钥的大小按位数计算。此前，用零字节填充的 1024 位密钥可以通过 WebDAV 令牌或 opkssh PK Token 的验证 |
| v0.17.0 | 一次安全审计的修复：一个共享不得包含另一个共享；访问文件（`authorized_keys_file`、`dsn_file` 等）不得位于共享之内；WebDAV 绝不会因为目录读取返回空而变成匿名；S3 遵守 `protocols`；30 秒内未通过身份验证的连接会被关闭；opkssh 会话在 `opkssh_max_age` 时结束；被撤销的会话保持被撤销并被关闭 |
| v0.17.1 | go-authn/oidc v0.2.4（重定向到 http 后不再接受密钥集）、servercert v0.3.0（ACME 缓存路径按 sshd 的 StrictModes 方式检查）、krl v0.5.0、revocation v0.3.0 |
| v0.17.2 | go-filesystems/s3 v0.3.0：S3 不再向未通过身份验证的调用方透露哪些访问密钥存在；预签名 URL 可用，最长有效一周，并在其自身日期之前被拒绝 |
| v0.18.0 | CI 固定使用 Go 1.27.1，而不是 `stable` |
| v0.18.1 | 当 `oidc` 块没有 `domains` 时，`fileshare check` 会发出警告：[提供方的裸名称会触及同名的本地账户]({{< relref "/configuration/identity.md" >}}) |
| v0.19.0 | go-authn/directory v0.11.0：`users "ldap"` 块会拒绝[向另一台机器发起的明文绑定]({{< relref "/configuration/identity.md" >}})；密码以摘要形式比较，且耗时恒定 |
| v0.20.0 | [`local_names`]({{< relref "/configuration/identity.md" >}})：只有 `oidc` 块明确说明时，提供方的名称才是本地名称（安全审计 F4） |
| v0.21.0 | [卷]({{< relref "/administration/volumes.md" >}})：特权的 `fileshare provisioner` 创建 ZFS 数据集、btrfs 子卷和 XFS/ext4 项目配额目录，管理 API 由它们提供共享；已满的目录共享在所有协议上给出同一个答案；管理套接字的 `allowed_uids` |
| v0.21.1 | [已满的共享]({{< relref "/administration/volumes.md#a-full-share" >}})在 SMB（`STATUS_DISK_FULL`）和 NFS（`NFS3ERR_NOSPC` / `NFS3ERR_DQUOT`）上会如实说明 |
| v0.22.0 | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}})：只有当 SSH 证书的域授权（`ssh-domain-grant@core.aai.geant.org`，EuroHPC SSH CA 配置）指明本主机时才会被接受，并以拒绝为默认；`ssh_accept_ungranted` 可放行没有授权的证书 |
| v0.22.1 | go-filesystems/sftp v0.5.1：[绑定到某个地址]({{< relref "/protocols/sftp.md#a-certificate-pinned-to-an-address" >}})的证书（`source-address`）在 `ssh_domains` 下，以及对提供方和 OpenPubkey 证书，都可从该地址被接受；此前它会在所有地址被拒绝 |
| v0.22.2 | 仅测试和文档：`source-address` 在 `ssh_domains` 下、对 OpenPubkey 证书，分别从 `::1` 和第二个 IPv4 地址（`127.0.0.2`）进行了测试；在 Linux 上，设置 `FILESHARE_REQUIRE_JUDGE` 时，缺少裁判会导致失败而不是跳过 |
| v0.23.0 | 通过 NFS、SMB 和 WebDAV，[btrfs 卷]({{< relref "/administration/volumes.md#the-size-a-client-sees" >}})显示其配额的大小（go-filesystems/nfs v0.7.0、smb v0.5.0、webdav v0.3.0 在每次查询时报告大小）；此前 btrfs 显示整个文件系统。共享来自管理 API 的服务器，在提供 NFS 且尚无共享时，启动时不再停止 |
| v0.24.0 | 更快的读取：目录或卷共享的 WebDAV GET 在纯 HTTP（不含 TLS）下通过 `sendfile(2)` 发送，NFS 以 1 MiB 读写（rtmax/wtmax），SMB 客户端每次请求可读取超过 64 KiB（`CAP_LARGE_MTU`）。在 CI 运行器上，256 MiB 的文件：WebDAV 6.6 GB/s，SMB 0.84，NFS 1.47（单个客户端） |
| v0.24.1 | 安全：空闲的 NFS 连接不再保留其最大调用所需的 3 MiB（24 个空闲连接曾占用 59 MiB）；SMB 帧头不再使服务器在字节到达之前就分配其声明的 8 MiB；SMB 请求不能再用一个信用额度换取 4 GiB（go-filesystems/nfs v0.8.1、smb v0.6.1 和 v0.6.2） |
| v0.25.0 | SFTP 应答 `limits@openssh.com`，因此 OpenSSH 的 `sftp` 每次最多读取 255 KiB（+25%）；一个会话最多持有 1024 个打开的句柄（go-filesystems/sftp v0.6.0） |
| v0.26.0 | 在每个具备此能力的协议上实现服务器端复制：WebDAV 的 `COPY`、SFTP 的 `copy-data`（OpenSSH 的 `sftp cp`）和 SMB 的 COPYCHUNK（资源管理器、Linux 的 `copy_file_range`）都在服务器上完成，通过主机文件之间的 `copy_file_range(2)`：在 btrfs 和 XFS 上是 reflink，在 CI 中实测 8 MiB 的复制占用 0 字节。安全：WebDAV 的 `COPY` 和每个 S3 `GET`/`HEAD` 不再把整个文件读入内存；S3 可流式传输任意大小的对象。更正：v0.24.0 的 WebDAV `GET` sendfile 仅发生在只读共享上，在可写共享上被一层包装隐藏了；自 v0.26.0 起两者都会发生 |
| v0.27.0 | ACME：证书**在启动时**申请，而不是在第一次连接时，失败后 1 分钟重试，间隔加倍直至 1 小时，同时服务器继续提供服务；**不发送 SNI** 的客户端（通过 IP 地址连接）会得到第一个域名的证书，而不是握手失败（go-authn/servercert v0.5.0）。[HARICA]({{< relref "/security/tls.md#harica-gant-tcs-which-account-and-caa" >}})：只有 Enterprise **Admin**（OV）账户可以无需质询签发，并且 CAA 必须允许 `harica.gr` |
| v0.28.0 | 管理 API 也[通过 HTTPS 为 OIDC 令牌]({{< relref "/administration/_index.md#over-https-for-oidc-tokens" >}})提供服务：Connect、gRPC-Web 和 gRPC 共用一个监听器，每个服务器一个受众，调用方按颁发者的主体或组指明，这是通向 Web 和原生界面的第一步 |
| v0.28.1 | 安全：使用 Go 1.27.2 和 golang.org/x/net v0.60.0 构建，修复了 GO-2026-6617（可能使服务器崩溃的 HTTP/2 HPACK 竞争）、GO-2026-6613、-6612、-6611、-6610 和 -6603。基于 HTTPS 的管理 API 使用 HTTP/2。`go install` 现在要求 Go 1.27.2，并会自动切换到它 |
| v0.29.1 | [重新加载]({{< relref "/administration/reload.md" >}})把目录服务中被清空的组视同被删除的组：它不授予任何权限，重新加载的说明会写明 `group @x has no members: it grants nothing`（go-authn/directory v0.12.0，它能区分空组与不存在的组） |
| v0.29.2 | **NetBSD（amd64）和 arm64 上 OpenBSD 的二进制文件**：fileshare 无法在 NetBSD 上构建，因为它经由 memguard 和 OpenPubkey 获得的 awnumar/memcall v0.1.2 使用了 `MADV_DONTDUMP`，而 NetBSD 没有这个标志。memguard v0.23.0 和 memcall v0.4.0 已在上游修复。现在每个发布平台都会在每个拉取请求上交叉构建。自 v0.29.1 起，每个发布版本都在其[发布页面](https://github.com/go-fileshare/fileshare/releases)提供各平台的静态二进制文件，附带 `SHA256SUMS` 和来源证明 |
| v0.29.0 | WebDAV：[对根目录的 `PROPFIND` 会列出各共享]({{< relref "/protocols/webdav.md#the-root-and-uploads-in-pieces" >}})（207；Depth infinity 为 403，写入为 405），因此文件管理器可以挂载整个服务器；上传可以分块进行，每个 `Content-Range` 追加到上一块结束的位置（go-filesystems/webdav v0.6.0），这正是[文件资源管理器]({{< relref "/administration/_index.md" >}})所需要的 |
| v0.30.0 | go-authn/directory v0.14.0：[`users "ldap"` 块]({{< relref "/configuration/identity.md" >}})默认读取 go-authn/authnd v0.11.0 所发布的内容：人员 `(\|(objectClass=posixAccount)(objectClass=account)(objectClass=uidObject))`，组 `(\|(objectClass=posixGroup)(objectClass=groupOfNames))`，`member` 的 DN 仅当它是人员搜索所返回的某人的 DN 时才计入。⛔ 面对机构自己的目录服务，默认会读取更多条目；显式设置的 `user_filter` 或 `group_filter` 不变 |
| v0.31.0 | [以每个人的身份提供目录服务]({{< relref "/configuration/as-each-person.md" >}})：`identity = "system"` 和 `identity` 块；以 root 启动的 `fileshare serve` 为每个人和每个共享启动一个工作进程，以其 uid、gid 和组运行，并以 `service_user` 提供各协议服务。仅限 SFTP 和 WebDAV；uid 0、低于 `min_uid` 的 uid 以及 `deny_uids` 会被拒绝 |

{{< callout type="warning" >}}
**升级到 v0.21.0**

**已满的目录共享**的应答方式不同。`EDQUOT`（配额）和
`ENOSPC` 现在是同一个错误，即“no space left on device (the share is full)”，
适用于每个目录共享，而不仅仅是卷：原先对 WebDAV 是
`500`、对 NFS 是 `NFS3ERR_IO` 的 `EDQUOT`，现在是 `507` 和 `NFS3ERR_NOSPC`，而 SFTP 的文本
就是那句话，不带路径。SMB 仍然应答拒绝访问。参见
[已满的共享]({{< relref "/administration/volumes.md#a-full-share" >}})。
{{< /callout >}}

{{< callout type="warning" >}}
**升级到 v0.20.0**

提供方的 `alice` 不再获得本地 `alice` 的共享。提供方的名称
就是其本地名称的站点，需要在 `oidc` 块中添加 `local_names = true`
（并设置 `domains`）；服务器会为每个因缺少它而被拒绝的名称记录
相应的日志行。
{{< /callout >}}

{{< callout type="warning" >}}
**升级到 v0.19.0**

指向另一台机器上 `ldap://` 且没有
`start_tls = true` 的 `users "ldap"` 块**不再能够启动**：请使用 `ldaps://`，或 StartTLS。
{{< /callout >}}

{{< callout type="warning" >}}
**升级到 v0.17.0**

一个共享位于另一个共享之内、两个共享使用同一来源，
或者访问文件（`authorized_keys_file`、`dsn_file`、`bind_password_file`、
`ssf` 的 `ca_file`）位于共享之内的配置**不再能够启动**；错误信息会指明
共享和路径。
{{< /callout >}}

{{< callout type="warning" >}}
**升级到 v0.9.0 之后的版本**

在其他机器可以访问的地址上，不使用 TLS 提供 WebDAV 且有人需要身份验证的配置
**不再能够启动**。请添加 `tls = true`
和 `tls` 块，或在终结 TLS 的代理之后使用 `plaintext = true`。参见
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})。
{{< /callout >}}

## 已验证 {#verified}

以下是让本项目未编写的客户端实际做到的事情，而不是
关于代码的断言。

- **macOS** 通过 SMB 挂载共享，同时 `curl` 通过 WebDAV 向同一个映像写入，
  并读回 WebDAV 所写的内容。Bob 通过 WebDAV 向其只能读取的共享写入会得到
  `403`；错误的密码得到 `401`。
- **go-smb2**（本项目未编写的客户端）在 CI 的 `-race` 下，
  通过 SMB 驱动二十个并发读取，同时有二十个通过 WebDAV 进行。
- 访问规则是**通过每个协议**检查的，而不仅仅是在
  配置中：alice 可以写入而 bob 不能，SMB 和 WebDAV 都是如此，
  而 NFS 根本不提供该共享。
- SFTP 证书已对照 **OpenSSH 自己的客户端**验证，当其证书被移开后，
  该客户端同样会拒绝这把密钥。
- 基于证书的 NFS 身份已在 go-filesystems/nfs 的 CI 中用**真实 Linux 客户端**
  （内核 6.17，ktls-utils 0.9）实测，[明文的 `MNT` 和 `tlshd` 的陷阱]({{< relref "/security/nfs-certificates.md#measured-with-a-real-linux-client" >}})
  就是这样发现的。
- KRL 未签名形式是一项实测：OpenSSH 9.6 和 10.3 读取已签名的
  KRL 时会跳过签名（go-authn/krl，对两者均已测试）。
- **go-authn/bridge 的互操作通道**（[#12](https://github.com/go-authn/bridge/pull/12)、
  [#17](https://github.com/go-authn/bridge/pull/17)）针对真实的 bridge 评判 fileshare v0.13.0：
  KRL、基于 NFS 的 CRL、通过共享信号撤销的 WebDAV 令牌、按作用域停用的 IdP，
  以及基于共享信号的 opkssh。

## 尚未实现 {#not-yet}

**通过 S3 写入。** `PUT` 和 `DELETE` 返回 403。库可以写入，但
某人只能读取的共享必须在与 SFTP 相同的位置拒绝，而这一点
尚未接入。拒绝胜过写了一半的对象。

**通过 S3 使用 OIDC 令牌。** 令牌只在 [WebDAV]({{< relref "/protocols/webdav.md" >}})上有效，
其他地方都不行。S3 的通常做法是 STS `AssumeRoleWithWebIdentity`，
它把令牌换成临时凭据，客户端随后用这些凭据签名，
这是与接受持有者令牌不同的机制，且尚未实现。

**带 `admin` 或 `metrics` 块的 `--isolate`。** 会被拒绝：没有任何单一
进程可供 API 变更应用。参见
[每个协议一个进程]({{< relref "/operations/isolation.md#not-with-an-admin-or-a-metrics-block-yet" >}})。

**在共享的 Linux NFS 客户端上，凭证书区分每个用户。** Linux
客户端把证书附加到挂载上，因此该挂载的每个用户都是
证书所指明的那个人。这是客户端的属性，那里的答案是
`sec=krb5`。参见[它无法承诺什么]({{< relref "/security/nfs-certificates.md#what-it-cannot-promise" >}})。

[S3 本身]({{< relref "/protocols/s3.md" >}})已经发布：共享就是存储桶，通过
SFTP 使用的同一棵每用户目录树提供。

## 实测，并如实陈述为实测 {#measured-and-stated-as-measured}

这些文档中属于保险或测量、而不是对缺陷的复现的说法，
在源码中也是这样写的：

- 在 `-race` 下，八个 goroutine 通过未包装的 fat32 驱动写入，
  目前不会产生竞争。[共享锁]({{< relref "/operations/locking.md" >}})是对契约的保险，
  `go-filesystems/interface` 对并发的基于路径的调用不作任何保证，而不是
  对已观察到的故障的修复。
- [构建标签的大小]({{< relref "/operations/build-tags.md" >}})是经过测量的，包括
  `hashicorp/go-plugin` 自身带来的 13.2 MB，这正是决定不采用
  插件框架的依据，也包括管理 API 引入 gRPC 之后的 11.7 MB，
  这正是 `nogrpc` 存在的原因。
- 曾经设计过自己的组标签证书扩展，并对照 Go 的 X.509 解析器进行了**测量**，
  该解析器拒绝了整个证书；因此组改为以
  [tag URI]({{< relref "/security/nfs-certificates.md#what-the-certificate-carries" >}})
  传递。
