---
title: "卷：fileshare 创建的存储"
weight: 10
description: "通过特权 fileshare 配给程序创建 ZFS、btrfs、XFS 和 ext4 存储，并由其提供共享服务。"
tags: [管理, 卷, zfs, btrfs, 配额]
---

自 v0.21.0 起，[管理 API]({{< relref "/administration/_index.md" >}})可以**创建共享所提供的存储**，
包括 **ZFS 数据集**、**btrfs 子卷**、**受项目配额限制的 XFS 或 ext4 目录**，
并设定大小、所有者且可被追踪，然后由其提供共享。
仅支持 Linux。尚不支持 Ceph。

创建这类存储需要 `CAP_SYS_ADMIN`（quotactl、btrfs 配额 ioctl、
`/dev/zfs`、mount(2)）。`fileshare serve` 要为陌生人解析五种网络协议，
因此不持有任何特权，具备特权的部分是同一二进制文件的第二种角色
`fileshare provisioner`，它位于 unix 套接字和一个小型封闭协议之后：

```
                         admin gRPC (unix socket 0600 + peer uid, or mTLS)
  operator ─────────────────────────────────────────────▶  fileshare serve
                                                             (unprivileged)
                                                                  │
                     provision gRPC (unix socket, SO_PEERCRED = fileshare's uid)
                                                                  ▼
                                                    fileshare provisioner
                                                    (CAP_SYS_ADMIN, CAP_CHOWN, CAP_FOWNER)
                                                                  │
                                    go-fsctl: zfs · btrfs · projquota (ioctls, no CLI)
```

使用一个二进制文件而不是两个守护进程：只需部署一个版本，只有一份协议
定义。存储本身由 [go-fsctl](https://go-fsctl.github.io/) 完成
（纯 Go，不使用 `zfs` 或 `btrfs` 命令）。设计以及构建过程中的变化
记录在 fileshare 仓库的
[`docs/volumes.md`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/docs/volumes.md) 中。

## 两个进程的配置 {#both-processes-configured}

`fileshare serve`，以用户 `fileshare`（uid 990）运行：

```hcl
# /etc/fileshare/fileshare.hcl
admin {
  listen          = "unix:///run/fileshare/admin.sock"
  state_file      = "/var/lib/fileshare/shares.json"
  source_roots    = ["/srv/fileshare/volumes"]
  provisioner     = "unix:///run/fileshare-provisioner/provisioner.sock"
  provisioner_uid = 0          # 必须在该套接字上应答的 uid；默认为 0
  allowed_uids    = [990]      # 可以调用此 API 的 uid
}
```

| 键 | |
|---|---|
| `provisioner` | 配给程序的套接字，即 `unix:///path`。没有它时，每个卷调用都返回 `FAILED_PRECONDITION`，也不会基于卷创建共享 |
| `provisioner_uid` | 在该套接字上应答的进程必须运行所用的 uid，每次连接时都从内核（`SO_PEERCRED`）读取：由他人绑定的套接字什么都得不到。未设置时为 0 |
| `allowed_uids` | 可选，仅限 unix 套接字：只有这些 uid 的调用会得到应答；其他任何调用返回 `PERMISSION_DENIED`，并记入审计。它只能**收窄**套接字 0600 模式所允许的范围（其所有者和 root），例如可使 root 无法调用。在 TCP 上会被拒绝，因为那里由客户端证书负责检查 |

卷的路径必须位于 `source_roots` 之下，与其他任何来源一样。

`fileshare provisioner`，以 root 运行：

```hcl
# /etc/fileshare-provisioner.hcl
provisioner {
  listen     = "unix:///run/fileshare-provisioner/provisioner.sock"
  client_uid = 990
  group      = "fileshare"
  max_volume = "10T"
  state_file = "/var/lib/fileshare-provisioner/volumes.json"

  parent "tank" {
    zfs  = "tank/fileshare"
    root = "/srv/fileshare/volumes/tank"
  }
  parent "fast" { btrfs = "/srv/fileshare/volumes/fast" }
  parent "plain" {
    xfs         = "/srv/fileshare/volumes/plain"
    project_ids = "100000-199999"
  }
}
```

| 键 | |
|---|---|
| `listen` | 只能是 `unix://<绝对路径>`。套接字的权限设为 0660，所有者为配给程序的用户（root）和 `<group>`，并且必须位于只有配给程序才能写入的目录中，即它**自己的**目录（`/run/fileshare-provisioner`），而不是 fileshare 的 |
| `client_uid` | 唯一会得到应答的 uid：fileshare 的 uid。**0 会被拒绝** |
| `group` | 拥有每个卷的根目录（`root:<group>`，模式 2770）。fileshare 必须属于该组：它通过该组写入卷，而不是以所有者身份写入，因为目录的所有者无需特权即可清除其项目 id，从而逃出 XFS/ext4 配额 |
| `max_volume` | 单个卷可以拥有的最大配额（`"10T"`、`"500G"`、`"1048576"`）；超过则返回 `OUT_OF_RANGE` |
| `state_file` | 此进程创建的内容（btrfs 子卷 id、XFS/ext4 项目 id），以便重启后它知道哪些是自己的 |
| `parent "<id>"` | 可以创建卷的位置，使用请求中指明的 id。`zfs`（卷作为其子数据集的数据集，`root` 为挂载位置）、`btrfs`、`xfs` 或 `ext4`（创建卷的目录）四者中恰好给出一个 |
| `project_ids` | XFS/ext4：此 parent 可以分配的项目 id 范围 |
| `enable_quota` | btrfs：当配额处于关闭状态时，允许配给程序将其开启。没有它时，配额关闭的 btrfs parent 会使启动中止，因为快照越多，qgroup 就会拖慢每次提交，而这是运维人员的选择 |

配给程序的文件中只有它的 `provisioner` 块：
若误把 fileshare 自己的文件交给它，会被拒绝，而不是只读取一半。

卷始终是 `<parent root>/<name>`：数据集 `tank/fileshare/<name>`
挂载于 `/srv/fileshare/volumes/tank/<name>`（带 `refquota`）、受其 qgroup 限制的
btrfs 子卷、带有来自该范围的项目 id 和硬性块限制的
XFS 或 ext4 目录。XFS 和 ext4 需要 `prjquota` 挂载
选项（ext4 还需要 `project,quota` 特性）。

## systemd 单元对 {#the-systemd-pair}

配给程序：

```ini
# fileshare-provisioner.service
[Service]
ExecStart=/usr/local/bin/fileshare provisioner -c /etc/fileshare-provisioner.hcl
RuntimeDirectory=fileshare-provisioner
RuntimeDirectoryMode=0755
StateDirectory=fileshare-provisioner
StateDirectoryMode=0700
CapabilityBoundingSet=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
# 以 root 运行，或以专用用户运行并设置：
# AmbientCapabilities=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
NoNewPrivileges=yes
SystemCallFilter=@system-service @mount quotactl quotactl_fd
SystemCallArchitectures=native
LockPersonality=yes
RestrictRealtime=yes
# 不要使用 ProtectSystem= 或 ReadWritePaths=：它们会让该服务拥有自己的
# 挂载命名空间，其挂载的 ZFS 卷将无人能看到。
# 也不要使用 Landlock：受 Landlock 限制的线程无法调用 mount(2)。
```

服务器，不带任何 capability：

```ini
# fileshare.service
[Unit]
Requires=fileshare-provisioner.service
After=fileshare-provisioner.service

[Service]
User=fileshare
Group=fileshare
ExecStart=/usr/local/bin/fileshare serve -c /etc/fileshare/fileshare.hcl
RuntimeDirectory=fileshare
StateDirectory=fileshare
# 不带任何 capability：CAP_SYS_RESOURCE 会让它越过 ext4 配额，
# 而 fileshare 拒绝在持有它的情况下提供卷服务。
CapabilityBoundingSet=
AmbientCapabilities=
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/srv/fileshare/volumes /var/lib/fileshare
```

服务器使用 `ProtectSystem=` 没有问题：执行挂载的是配给程序，
而挂载会传播进服务器的命名空间，而不是传播出去，因此在服务器启动后
挂载的 ZFS 卷对它是可见的。

{{< callout type="info" >}}
**这些是示例，不是经过测试的单元**

这两个单元都没有在 CI 中运行过：端到端任务在 `sudo` 下
运行这两个进程。
{{< /callout >}}

{{< callout type="warning" >}}
**请先升级配给程序**

配给程序会拒绝带有其版本未定义字段的消息，并关闭该连接，
因此 `fileshare serve` 和
`fileshare provisioner` 必须运行相同的版本。
{{< /callout >}}

## 管理 API 的卷调用 {#the-admin-apis-volume-calls}

`fileshare serve` 将这些调用转发给配给程序
（[`admin.proto`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto)）：

| | |
|---|---|
| `ListParents` | 可以创建卷的位置：每个 parent 的 id、种类、根目录、可用和总空间、是否支持快照；以及 `max_volume_bytes` |
| `CreateVolume` | 配额为 `quota_bytes` 的卷（0 会被拒绝）。按名称幂等：相同的配额会返回已存在的卷（`created` 为 false），不同的配额则返回 `ALREADY_EXISTS` |
| `ResizeVolume` | 修改配额，增减皆可，但不得低于已使用量（`FAILED_PRECONDITION`） |
| `SnapshotVolume` | 只读的命名快照：ZFS 和 btrfs 支持；XFS 和 ext4 返回 `UNIMPLEMENTED` |
| `GetVolume`、`ListVolumes` | 每个卷，包括其配额、使用量、快照以及**使用它的共享** |
| `DeleteVolume` | 有共享使用时会被拒绝，请先删除共享。含有数据或快照的卷需要 `destroy_data` |
| 带 `volume { parent, name }` 的 `CreateShare` | 以目录形式由该卷提供的共享 |

`DeleteShare` 绝不会删除卷。“使用它的共享”包括
由它创建的共享**以及**任何其映像或目录位于其内部的共享，
也包括配置文件中的共享，无论处于已提供服务、已停用还是不可用状态。

名称须匹配 `^[a-z0-9][a-z0-9_-]{0,62}$`，请求绝不携带
路径：它指明的是配给程序自己配置中的某个 `parent`。

在 `admin` 块中设置 `reflection = true` 后，`grpcurl` 就可以发起这些调用。
套接字权限为 0600，而上文的 `allowed_uids` 只列出 fileshare 的 uid，因此请
以该用户运行：

```sh
S=/run/fileshare/admin.sock
A=fileshare.admin.v1.AdminService

sudo -u fileshare grpcurl -plaintext -unix $S $A/ListParents

# XFS parent 上的一个 32 GiB 卷
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "34359738368"}' \
  $A/CreateVolume

# 由它提供服务的共享
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"name": "projects",
       "volume": {"parent": "plain", "name": "projects"},
       "grants": [{"subject": {"group": "staff"}, "access": "ACCESS_WRITE"}]}' \
  $A/CreateShare

# 扩容到 64 GiB
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "68719476736"}' \
  $A/ResizeVolume
```

### 状态码 {#status-codes}

配给程序的状态码原样返回：`INVALID_ARGUMENT`（名称
不符合语法、未知的 parent、配额为 0）、`OUT_OF_RANGE`（超过
`max_volume`）、`RESOURCE_EXHAUSTED`（配额或调整大小所增加的部分
超过 parent 当前的可用空间；或项目 id 范围已用尽）、
`ALREADY_EXISTS`、`NOT_FOUND`、`FAILED_PRECONDITION`、`UNIMPLEMENTED`。

有三个是 fileshare 自己的：

- 未配置 `provisioner`：`FAILED_PRECONDITION`；
- 配给程序拒绝此服务器的 uid：同样是 `FAILED_PRECONDITION`，因为这是
  部署错误（其 `client_uid` 不对），而不是管理
  调用方无权执行的操作；
- 配给程序没有应答：`UNAVAILABLE`。

## 提供卷服务之前检查什么 {#what-is-checked-before-a-volume-is-served}

fileshare 向配给程序询问卷的位置，然后自行检查：

- 路径是干净的绝对路径，并且解析之后位于 `source_roots` 之下；
- 它是一个目录；
- `statfs` 报告的是该种类对应的文件系统（`ZFS_SUPER_MAGIC`、
  `BTRFS_SUPER_MAGIC`、`XFS_SUPER_MAGIC`、`EXT4_SUPER_MAGIC`）；
- **ZFS**：它是挂载点，即其设备不同于其父目录的设备。未挂载的
  数据集的目录会在没有任何配额的情况下接受写入。（挂载在那里的是哪个数据集
  则不做验证：配给程序的应答没有说明。）
- **btrfs**：它是子卷的根，因为 parent 下的普通目录
  没有 qgroup；
- **XFS 和 ext4**：它带有非零的项目 id，且其新文件会
  继承该 id，因为不属于任何项目的目录会在无限制的情况下被提供服务。

状态文件保存的是卷的**名称**，而不是路径，每次启动都会重新询问并
检查。如果卷已不存在、未通过检查，或其配给程序
在 10 秒内没有应答，则其共享会保持**已定义但不提供服务**，这一点会在
启动时、通过 [`fileshare check`]({{< relref "/configuration/check.md" >}})，以及在
共享的 `unavailable` 字段中说明，而服务器的其余部分照常启动。
`EnableShare` 会再次尝试。

{{< callout type="error" >}}
**`fileshare serve` 拒绝以 root 身份或带 `CAP_SYS_RESOURCE` 提供卷服务**

ext4 允许二者越过项目配额写入（fs/quota/dquot.c，
`ignore_hardlimit`）：在 go-fsctl/projquota 的 CI 中，root 向一个 8 MiB 的项目写入了 16 MiB。
XFS 没有这样的豁免，但这条规则对进程是统一的，适用于每一种类：
fileshare 读取其有效 uid 以及 `/proc/self/status` 中的
`CapEff`/`CapPrm`（被许可的 capability 离生效只差一次
`capset(2)`），读不到状态则视为
持有该权限。它拒绝的是**提供服务**，而不是卷调用：以 root 创建存储
无害，以 root 向其中写入则不然。`fileshare check` 会指出
是哪一种情况。因此配给程序会拒绝 `client_uid = 0`，
而上面的服务器单元的 `CapabilityBoundingSet=` 为空。
{{< /callout >}}

## 已满的共享 {#a-full-share}

XFS 用 `ENOSPC` 表示项目已满；ext4、btrfs 和 ZFS 用 `EDQUOT`。自
v0.21.1 起，每个协议都会应答“已满”：

| 协议 | 无剩余空间（`ENOSPC`） | 配额（`EDQUOT`） |
|---|---|---|
| WebDAV | `507 Insufficient Storage` | `507 Insufficient Storage` |
| NFS | `NFS3ERR_NOSPC` (28) | `NFS3ERR_DQUOT` (69)，与 RFC 1813 和 Linux 的 nfsd 一致 |
| SFTP | `SSH_FX_FAILURE`，“no space left on device (the share is full)”，因为版本 3 没有对应的代码 | 相同 |
| SMB | `STATUS_DISK_FULL` (0xC000007F) | `STATUS_DISK_FULL`，与 Samba 一致（“Windows apps need this, not NT_STATUS_QUOTA_EXCEEDED”） |
| S3 | 以只读方式提供服务 | 以只读方式提供服务 |

在 v0.21.0 中，SMB 返回 `STATUS_ACCESS_DENIED`，NFS 对配额
返回 `NFS3ERR_NOSPC`。

这适用于每个[目录共享]({{< relref "/configuration/shares.md" >}})，而不仅限于
卷，参见[升级说明]({{< relref "/status.md" >}})。

## 客户端看到的大小 {#the-size-a-client-sees}

每个协议在每次查询时都会向服务器询问共享的大小：NFS 的
`FSSTAT`、SMB 的 `FileFsFullSizeInformation` 以及 WebDAV 的 RFC 4331 配额
属性。

- **ZFS、XFS 和 ext4**：大小就是 `statfs` 在卷内部所报告的值，
  内核以配额作答（ZFS 为 `refquota`，XFS 和 ext4 为项目
  配额）。
- **btrfs**（自 fileshare v0.23.0 起）：btrfs 的 `statfs` 忽略 qgroup，
  报告的是整个文件系统，因此服务器改用配给程序提供的数字。
  大小是卷的配额，可用空间是配额减去 qgroup 的已引用字节数，
  或者在文件系统自身的可用空间更少时取后者。服务器在后台向配给程序询问，
  客户端查询期间最多每 5 秒一次，其间以最近一次的
  数字作答。btrfs 在提交事务时更新 qgroup 的计数
  （默认每 30 秒一次），因此客户端看到的可用空间
  可能比写入滞后这么久。在 v0.23.0 之前，btrfs 卷显示的是
  整个文件系统；配额依然有效。

已用空间来自配给程序（`used_bytes`）：读取 XFS 或 ext4
项目的用量需要 `CAP_SYS_ADMIN`，而 fileshare 没有这一权限。

## 配给程序的承诺 {#what-the-provisioner-promises}

- **一个 uid，从内核读取。** 对端在连接时通过 `SO_PEERCRED` 识别，
  除 `client_uid` 之外的任何人（包括 root）在请求的任何一个字节被解码之前
  就会得到 `PERMISSION_DENIED`。
- **封闭的动词集合。** 其服务之外的方法，或无法解码、
  或带有其未定义字段的消息，不会得到应答：
  连接被关闭。
- **绝不触碰并非自己创建的内容。** ZFS：数据集自身的
  `fileshare:volume` 属性，设置在数据集上，因为用户属性
  会被继承，而继承来的标记不是它自己的。btrfs：记录在 `state_file`
  中的子卷 id 和 uuid。XFS/ext4：位于 parent 范围内的项目 id，
  *并且*已被记录。在每次调整大小、快照和删除之前都会检查。
- **删除会拒绝数据和快照**，除非设置了 `destroy_data`，并且
  在执行之前就会记入日志。

## 实测 {#measured}

在 fileshare 的 CI 中，以 root 身份在真实存储池和回环文件系统上运行，`fileshare serve` 以 `nobody` 运行并通过
管理 API 驱动：经 WebDAV 和 SFTP 的写入都会在 32 MiB 卷的配额处停止，
四种种类均如此。

## 尚未实现 {#not-yet}

- **Ceph。** CephFS 配额和 RBD 留待后续阶段。
