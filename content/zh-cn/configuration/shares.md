---
title: "共享：映像、设备、目录"
weight: 10
description: "共享可以提供什么（磁盘映像、设备或主机目录），以及如何选择其文件系统和分区。"
tags: [配置, 共享]
---

共享提供以下三者之一：**磁盘映像**、**设备**节点，或
**主机的目录**。映像和设备带有本程序自行读取的文件系统；
目录则是主机自己的文件系统。

## 映像及其内部的文件系统 {#an-image-and-the-filesystem-inside-it}

映像内部的文件系统是**推断出来的，而不是声明的**。
[`go-filesystems/detect`](https://github.com/go-filesystems/detect) 读取
魔数，并返回拥有它的驱动：**fat32、exfat、ext4、ntfs、ufs、
iso9660、squashfs 或 hfsplus**，每个驱动都是同一种形式，即
`OpenReader(io.ReaderAt, int64)`，文件系统位于偏移量零处。

## 无法通过嗅探识别的四种 {#the-four-that-cannot-be-sniffed}

**apfs、btrfs、xfs 和 zfs** 都打开*磁盘*映像并选择一个**分区**，
因此在偏移量零处找不到魔数。共享需要指明是哪一个：

```hcl
share "photos" {
  image      = "/srv/disk.img"
  filesystem = "xfs"
  partition  = 2       # 也可以省略：驱动会选取第一个数据分区
}
```

## 任何文件系统的分区 {#a-partition-for-any-filesystem}

含有 FAT32、ext4 或 exFAT 的磁盘映像，与含有 XFS 的映像一样常常
在前面有分区表，而检测读取的是偏移量零处，分区映像的
该处是*分区表*。因此先选择分区，
再把该分区的视图交给驱动：

```hcl
share "photos" {
  image           = "/srv/disk.img"
  partition_label = "photos"          # lsblk 所称的 PARTLABEL
}
```

指定分区有三种方式，并且**只能给出一种**，因为两种不一致的方式会
提供代码先尝试的那一个：

| | |
|---|---|
| `partition = 2` | 从 1 开始计数，与分区工具的显示方式一致 |
| `partition_label = "photos"` | GPT 分区名称 |
| `partition_uuid = "…"` | GPT 唯一 GUID，即 Linux 的 `PARTUUID` |

{{< callout type="error" >}}
**索引会移动**

磁盘被重新分区、某个工具以另一种顺序写入条目、映像
从少一个分区的备份中恢复，`partition = 2` 就会**悄悄地**指向
别的分区，因为那里仍然能找到文件系统。标签或
UUID 指向的是分区本身，这就是 fstab 多年前就不再使用索引的原因。
保留索引是为了 MBR 映像，它们两者都没有。
{{< /callout >}}

选择了分区的共享是**只读**的，并会在启动时说明：
驱动的偏移量是相对于分区的，而底层的文件是整个
磁盘，因此写入会落在从*映像*开头起的该偏移量处，也就是
常常落在分区表上。

## 指定文件系统会关闭检测 {#naming-a-filesystem-turns-detection-off}

{{< callout type="error" >}}
**这正是它的意义，也是它的风险**

映像会被当作*那种*文件系统打开，否则被拒绝。被告知是 XFS 的 FAT32 映像
不会变成 XFS 共享；它会启动失败，并说明被要求
按什么方式打开它。
{{< /callout >}}

对于可以嗅探的文件系统，`filesystem` 同样可用，此时会对二者进行
比较：在 FAT32 映像上声明 `ext4` 的共享会被拒绝，并提示 *the
share says ext4 and the image holds fat32*。站点就是这样拒绝
误检测，而不是事后才发现。
[`check`]({{< relref "/configuration/check.md" >}}) 会用星号标记指定的驱动，因为那一行
不是识别出来的，而是断言出来的。

## 设备，而不仅仅是映像 {#a-device-not-only-an-image}

```hcl
share "photos" {
  image = "/dev/sda"        # 设备节点，而不是文件
}
```

**不涉及任何特权。** 这里不挂载任何东西，
[`go-filesystems`](https://github.com/go-filesystems) 驱动在用户空间中读取 ext4、xfs 以及
其他文件系统，因此没有 `mount(2)`，也就不需要
`CAP_SYS_ADMIN`。打开设备是一次普通的 `open(2)`，由该节点上的
权限控制：

| | 所有者 | 模式 | 足够的条件 |
|---|---|---|---|
| Linux `/dev/sda` | `root:disk` | 0660 | 属于 `disk` 组 |
| macOS `/dev/disk0` | `root:operator` | 0640 | 属于 `operator` 组，并具有完全磁盘访问权限 |

拒绝信息会指出需要哪个组，而不是 `permission denied`，
答案绝不是 `sudo`。

⛔ **设备共享是只读的**，原因与选择了分区的共享相同：
向正在使用的磁盘写入，不是可以代替你做出的决定。

{{< callout type="error" >}}
**设备以独占方式打开，这关乎正确性，而不是谨慎**

如果内核已从该设备挂载了文件系统，文件系统的页缓存
持有比设备更新的元数据，因此原始读取者会看到更新之前的
目录块和更新之后的 inode 块并存，
这是磁盘上任何时刻都从未存在过的状态。块设备上的 `O_EXCL` 是内核
自己用于表示“其他任何人都不行，包括挂载”的原语；
`mkfs` 和 `fsck` 使用的就是它。正在使用的设备会被拒绝，
并指明设备名称和原因。
{{< /callout >}}

有两点是经过实测而不是想当然的：

- 设备的 `Stat().Size()` 为 **0**，因此需要向设备本身询问长度。
  在 Linux 上，定位到末尾会给出答案，而在 macOS 上会**返回 0 且没有错误**，
  所以 Darwin 改用 `DKIOCGETBLOCKCOUNT`。
- **原始**节点（`/dev/rdiskN`，或 Linux 上的 `O_DIRECT`）会拒绝未对齐的
  读取，而读取偏移量 11 处的两字节字段正是解析 FAT BPB
  的内容。对设备的读取会向外取整到完整的块；没有这一步，
  `/dev/rdisk4` 会报告 `unknown filesystem`。

## 目录，而不仅仅是映像 {#a-directory-not-only-an-image}

自 v0.8.0 起，共享可以提供主机的目录（例如容器的卷）
而不是映像：

```hcl
share "photos" {
  directory = "/data/photos"
  allow     = ["@family"]
}
```

它使用 [go-filesystems/osfs](https://github.com/go-filesystems/osfs)，通过 **`os.Root`**
访问目录树：向上逃出的 `..`、绝对
路径以及指向外部的符号链接，都由内核支持的根目录拒绝，
而不是靠字符串测试。客户端可以*创建*指向 `/etc` 的链接，但没有任何东西会
跟随它。植入目录树中的 FIFO 会被拒绝，而不是任由其使
服务器挂起。

{{< callout type="error" >}}
**一个共享不得包含另一个共享（自 v0.17.0 起）**

目录树中包含另一个共享的映像或目录的目录共享，
会在**启动时被拒绝**，通过管理 API 创建时同样如此：能使用外层
共享的人将可以读写内层共享而无需被允许，
包括匿名 NFS。同一来源上的两个共享也是如此，唯一的例外是同一磁盘映像的
两个不同分区。
{{< /callout >}}

{{< callout type="info" >}}
**不受映像锁约束**

映像驱动拥有一个文件，并且对同时发生的两次调用不作任何保证，
这就是[每个映像都被包裹在一把锁中]({{< relref "/operations/locking.md" >}})的原因。
主机目录树归内核所有，内核只对必须串行化的操作按次串行化，
仅此而已；在每个客户端的每个文件前加一把互斥锁，
会是文件服务器所能做的最慢的事情。
{{< /callout >}}

⛔ 共享有 `image` **或** `directory`，不能两者都有，也不能两者都没有。
`filesystem` 和 `partition` 用于映像：指明了其中之一的目录共享会
被拒绝，因为这意味着该共享本应是映像。
`read_only = true` 对目录和映像同样有效；同时声明它
又列出 `writers` 的共享会被拒绝，因为 `read_only` 会占上风。

告知客户端的容量是目录树所在文件系统的容量；无法提供该信息的
平台会将其保留为零，协议会把它理解为
“未知”而不是“已满”。
