---
title: "以每个人的身份提供目录服务"
weight: 15
description: "identity = \"system\"：目录共享的文件由以登录者身份运行的工作进程打开，因此内核会应用目录树的权限，而该用户创建的内容归该用户所有。"
tags: [配置, 共享, 身份, 安全]
---

通常，一个 fileshare 进程以同一个账户为所有人提供服务：fileshare
决定谁可以做什么，它创建的每个文件都属于该账户。
目录共享也可以改为**以每个人的身份**提供服务。每位登录者
的文件都由以其 uid、gid 和
组运行的进程打开，因此内核会应用目录树的权限，而该用户创建的内容
归该用户所有。

```hcl
identity {
  service_user = "fileshare"   # 以其身份提供各协议服务的账户
  min_uid      = 1000          # 默认值；无论此处如何设置，uid 0 都会被拒绝
  min_gid      = 1000
  deny_uids    = [65534]
  idle         = "2m"          # 空闲的工作进程保留多久
}

share "home" {
  directory = "/srv/home"
  identity  = "system"
}
```

## 三个进程 {#three-processes}

`fileshare serve` 必须**以 root 启动**。它保持 root 身份只是为了启动
每个人的工作进程。所有面向网络的部分都运行在第二个
进程中，以 `service_user` 身份运行。

| 进程 | 运行身份 | 职责 |
|---|---|---|
| **启动器**（`fileshare serve`） | root | 绑定监听器，启动前端，在前端请求时启动工作进程。它不使用任何协议。 |
| **前端** | `service_user`，带有该账户的组 | 使用各协议并进行身份验证，一如 `serve` 一贯所做。对于这样的共享，它向启动器请求的是**一个人和一个共享**，绝不是 uid 或路径。 |
| **工作进程** | 该用户本人 | 通过启动器移交的套接字，向前端提供一个共享 |

工作进程首先检查自己是否恰好以预期的 uid、gid 和组运行，
以及 `setuid(0)` 是否失败。只有这样，它才会通过 `os.Root` 打开
目录树。一个工作进程为该人对该共享的所有连接提供服务，
SFTP 和 WebDAV 都一样，并且在 `idle` 时间内无人使用后就会停止。
服务器停止时，工作进程也随之停止。

## 一个人在本系统中是谁 {#who-a-person-is-on-this-system}

启动器**自行**解析此人，顺序如下：

1. **目录服务**，如果其中带有 `uidNumber` 和 `gidNumber`：
   - `user` 块的 `attributes = { uidNumber = "2001", gidNumber = "2001" }`；
   - 带有 `attributes = ["uidNumber", "gidNumber"]` 的 LDAP 来源；
   - SQL 来源的 `users_attributes` 查询。

   附加组是列出此人的组（`group`
   块的 `members`、`memberUid`）以及此人所指明的组（`memberOf`），
   各自以其 `gidNumber` 标识。没有 `gidNumber` 的组不是系统组。
2. **`/etc/passwd` 和 `/etc/group`**，按名称查找。

**被拒绝的情况**：uid 0、gid 0、`(uid_t)-1`、低于 `min_uid` 的 uid、低于
`min_gid` 的主 gid，以及 `deny_uids` 中的每个 uid。低于
`min_gid` 的附加组会被忽略，日志中会说明。在这两处都找不到的人
同样会被拒绝：不存在共享的后备账户。由身份
提供方担保的人，只有配合
[`local_names`]({{< relref "/configuration/identity.md" >}})（它使
其名称成为本地名称）才会获得工作进程。

因此，被接管的前端只能为存在且被允许的人，
在存在的共享上提出请求。它仍然负责判定**谁**已通过
身份验证，而启动器无法替它检查这一点。

## 客户端看到什么 {#what-a-client-sees}

| | SFTP | WebDAV |
|---|---|---|
| 获取工作进程的时机 | 登录时，覆盖整个连接 | 每个请求一次；锁与工作进程存活时间一样长 |
| 启动器拒绝此人 | 共享不在其目录树中（目录树中空无一物则登录被拒绝） | `403` |
| 无法启动任何工作进程 | 同上 | `503` |

原因记入服务器日志，绝不会告知客户端。

{{< callout type="warning" >}}
**目前不支持 SMB、NFS 或 S3**

它们会拒绝这样的共享，[`check`]({{< relref "/configuration/check.md" >}})
会指出这一点。SMB 和 NFS 需要其库中为每个会话提供一个文件系统。NFS
只会在 Kerberos 下才支持，因为 `AUTH_UNIX` 是客户端自己的
声明（RFC 5531 §14）。
{{< /callout >}}

## 它对系统有什么要求 {#what-it-asks-of-the-system}

- **Unix**：Linux、BSD 和 macOS。在 Windows 上该共享会被拒绝。
- **`service_user` 可读**：配置以及它所指明的每个文件
  （密码文件、主机密钥、证书），因为前端要读取它们。
- **用户可执行**：二进制文件，因为每个工作进程都以
  各自用户的身份运行它。
- 添加或更改这样的共享需要**重启**。启动器只读取一次
  配置，并拒绝它不认识的共享，而不是让
  前端去提供它。
- `--isolate` 不能与之同时使用。

设计以及每项选择的理由，见
[docs/identity.md](https://github.com/go-fileshare/fileshare/blob/main/docs/identity.md)。
