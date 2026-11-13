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

## HFS+ 镜像默认只读，除非共享选择启用写入 {#hfsplus-images-are-writable}

自 v0.44.0（hfsplus v0.9.0）起，镜像中含 HFS+ 的共享可以像 FAT、exFAT 和 ext4 一样以可写方式打开：
客户端可以创建、修改和删除文件，它所设置的时间、权限模式和所有者也会被存储。
此前 HFS+ 镜像以只读方式打开。

**自 v0.45.1 起默认关闭，现在仍然关闭。** hfsplus v0.9.0 不能抵御崩溃（它把整个镜像放在内存中，
每次操作都重写整个镜像，并且从不把卷标记为使用中），所以 v0.45.1 关闭了写入（v0.44.0 和 v0.45.0
默认会写入，最大 256 MiB）。**自 v0.46.0（hfsplus v0.10.0）起**，可写的 HFS+ 卷**原地写入**：

- **没有内存开销。** 镜像不会被复制到内存中（只有 8 MiB 的读缓存）；每次提交只写入发生变化的扇区。
  64 GiB 的上限也已取消。
- **崩溃是可见的。** 每次提交都会把卷标记为使用中：已卸载位和日志位在第一次写入头部时被清除，
  结束时恢复，所以 `fsck_hfs -q` 能发现崩溃，带日志的卷也一样。
- **写入失败会让卷停止。** 刷新失败后，卷会拒绝写入（`ErrVolumeFailed`），直到共享重新打开。

它仍然需要**主动选择启用**，因为 hfsplus 还有未完成的事项：持久的小写入每次提交需要 3 次 fsync，
大约比 macOS 慢 3 倍（go-filesystems/hfsplus#35）；崩溃测试尚未加固（go-filesystems/hfsplus#37）；
崩溃后，带日志的卷会在没有日志的情况下挂载，直到运行 `diskutil enableJournal`。这些问题修复后，
默认值可以恢复。请先备份镜像。共享通过 `hfsplus_writable_max` 选择启用，默认 `"0"`
（表示从不；可以是字节数，或带 K、M、G、T 的数）：

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"   # 启用写入
}
```

自 v0.46.0 起，这个大小不再限制内存。它仍然是一个大小，这样为 v0.44.0 至 v0.45.1 编写的配置含义不变
（写着 `"256M"` 的共享不会开始写入 2 TiB 的镜像）；像 `"1T"` 这样的大值就是单纯的启用。

使用默认值、超过该大小或大小未知时，共享以**只读**方式打开（范围读取和时间仍然可用），
服务器会写出一行，说明镜像和原因，例如没有任何设置时：

```text
/srv/mac.img is an HFS+ volume and hfsplus_writable_max is 0, the default: HFS+ images are written only when a share sets a size, because a durable small write is about 3x slower than on macOS (go-filesystems/hfsplus#35), the crash tests are not finished (go-filesystems/hfsplus#37), and after a crash a journaled volume mounts without its journal until diskutil enableJournal, so mac is read-only
```

超过该大小时：

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes), so mac is read-only
```

**重新加载会跟随该设置（自 v0.46.0 起）。** 当一次重新加载改变了 HFS+ 镜像是否被写入
（设置被移除、添加，或跨过镜像大小），镜像会被重新打开：移除启用设置会让共享立即变为只读，
添加则让它变为可写。v0.46.0 之前会保留已打开的驱动程序，所以共享在移除启用设置后仍然可写，
直到服务器重启。从一个可写大小改为另一个可写大小不会有任何变化。

**带日志**的卷（由 `hdiutil create -fs "Journaled HFS+"` 或 macOS 创建）只有在其日志为空时才会被写入，
正常卸载后日志即为空。含有事务的日志，或无法验证的日志，会被 macOS 重放并覆盖服务器写入的内容，
因此驱动程序会拒绝它，**共享改为只读打开**。服务器照常启动；打开共享时它会写出一行，
说明镜像、共享和原因：

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

要让这样的共享可写，需要先让日志被重放：在 Mac 上挂载该镜像并正常推出，或对其运行 `fsck_hfs`。
写入带日志的卷期间，它的日志位是被清除的；崩溃后它仍保持清除，所以该卷会在没有日志的情况下挂载，
直到运行 `diskutil enableJournal`。设置了 `read_only = true` 的共享、设备和分区从不尝试以可写方式打开。

卷通过共享自己的文件描述符写入（hfsplus v0.11.0）：就是已经受限并加锁的那个文件，绝不会按名称重新打开镜像。
与卷不匹配的大小会让共享以只读方式打开，并写出一行说明。

**另一个写入者。** 同一镜像上的第二个写入者会损坏镜像。可写方式打开时会对镜像加一个排他的
建议性锁 `flock`（Linux、macOS 和 BSD）；如果另一个进程持有它，例如同一镜像上的第二个 fileshare，
共享就以只读方式打开并说明原因。该锁是建议性的：macOS 的挂载不会加这个锁，其他平台则没有锁。

⛔ **不要写入已挂载的镜像**，无论是用本服务器还是其他东西：双方都不知道对方的修改。
请提供没有其他程序打开的镜像。

## 操作系统已挂载的可写镜像以只读方式提供 {#an-image-the-os-has-mounted}

自 v0.45.0 起，在以写入方式打开**任何**可写镜像共享（FAT32、exFAT、ext4、UFS、btrfs、XFS、ZFS、HFS+，无论驱动是什么）之前，服务器会询问操作系统是否已附加或挂载该镜像。如果是，该共享以**只读**方式打开并被标记为只读，服务器会写出一行日志，指明镜像、操作系统的设备和挂载点以及共享名称：

```text
/srv/disk.img is in use by the OS (/dev/loop3, mounted at /mnt/disk), and a second writer would corrupt it, so disk is read-only
```

上文所述的建议性 `flock` 无法做到这一点：操作系统挂载镜像时不会加锁。镜像按设备号和 inode 匹配，绝不比较路径字符串，因此指向已挂载文件的符号链接或硬链接同样会被发现。

| 操作系统 | 查询方式 |
| --- | --- |
| Linux | 每个回环设备的 `/sys/block/loop*/loop/backing_file`（`mount -o loop`、`losetup` 和 udisks 创建的设备）；挂载点来自 `/proc/self/mountinfo`。已附加的回环设备即使上面没有挂载任何内容也算数。 |
| macOS | `hdiutil info -plist`，即每个已附加镜像的 `image-path`，每次加载调用一次，超时 15 秒。用 `-nomount` 附加的镜像也算数。 |
| FreeBSD、其他 BSD、Windows | **无法检测。** 共享与以前一样可写。 |

**检查本身失败时**（容器中没有 `/sys/block`、`hdiutil` 缺失或超时、输出不是属性列表），没有人知道镜像是否已挂载。由 `image_mount_unknown` 决定：

```hcl
share "disk" {
  image               = "/srv/disk.img"
  image_mount_unknown = "read_only"   # 默认值；"write" 仍以可写方式提供
}
```

默认值是 `read_only`：只读共享是一种可见的状态，日志行会指明原因，人可以去修复；挂载之下的可写共享会损坏双方的数据，而且要到事后才会有人发现。`write` 适用于检查无法工作的主机（没有 sysfs 的容器）且没有人挂载镜像的情况。已知被操作系统挂载的镜像无论此设置如何都是只读的。检查无法读取的后备路径（权限被拒绝、符号链接循环）被视为“无法判断”，而不是“另一个镜像”：它同样交由此设置处理。只有不存在的路径才会被跳过。在 Linux 上，后备文件已被删除的回环设备，如果进程可以打开 `/dev/loopN`，则通过内核自己的记录（`LOOP_GET_STATUS64`）进行匹配，否则无法匹配。

**检查看不到的情况**

- **共享打开镜像之后才发生的挂载。** 检查在打开镜像时运行：启动时、通过管理 API 创建共享时，以及重新加载重建共享（人员或列表发生变化）并发现镜像在此期间已被挂载时；此时该共享会得到自己的只读打开。没有任何变化的重新加载不会再次检查，也没有任何机制监视之后的 `hdiutil attach` 或 `mount -o loop`。在挂载一个以可写方式提供的镜像之前，请先停止该共享或重启服务器。
- **不经过回环设备或 `hdiutil` 的挂载：** `qemu-nbd -c`（nbd 设备）、FUSE 挂载（`fuse2fs`、`guestmount`），以及打开着该镜像的虚拟机或其他程序。
- 其他操作系统（见表格），以及在另一个挂载命名空间中附加的回环设备。
- 因挂载而变为只读的共享，即使挂载已消失，重新加载后仍保持只读；请重启它以再次以写入方式打开。

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

## SMB 如何处理无法存储的时间 {#what-smb-does-with-a-time-it-cannot-store}

客户端设置了存储无法保留的时间时，默认会被**忽略**。自 v0.42.0 起，共享可以用
`unstorable_times` 来选择。无法存储的有：与所报告的值不同的 CreationTime、单独
发送的 ChangeTime，以及对驱动程序无法设置时间的镜像（NTFS、APFS、ISO 9660、SquashFS）设置的访问时间或修改时间
（主机目录以及 FAT、exFAT、ext4 和 HFS+ 镜像可以存储它们）：

```hcl
share "photos" {
  directory        = "/data/photos"
  unstorable_times = "log"      # "ignore"（默认）、"refuse" 或 "log"
}
```

`refuse` 会向客户端返回错误，`log` 会写一行，写明共享和路径。
按共享类型划分的确切情形见
[真实的文件时间]({{< relref "/protocols/times.md#when-unstorable" >}})。
