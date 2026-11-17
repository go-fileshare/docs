---
title: "automount"
weight: 10
description: "无需密码对话框，在操作系统的文件管理器中挂载 SMB 共享，并可卸载。"
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.2
**无需任何密码对话框**，即可在操作系统的文件管理器中挂载 SMB 共享，并可卸载。
它是建立在三个现有后端之上的纯 Go（无 cgo）小型门面。本页根据其
[README](https://github.com/go-fileshare/automount/blob/main/README.md)
编写；与仓库不一致之处，以仓库为准。

| 系统 | 后端 | 出现的内容 |
|---|---|---|
| macOS | [go-macos/netfs](https://github.com/go-macos/netfs)（NetFS，`NoUI`） | `/Volumes` 下的卷，位于 Finder 侧边栏 |
| Linux（GNOME） | [go-freedesktop/gvfs](https://github.com/go-freedesktop/gvfs)（应答 GVfs 自身的 `AskPassword`） | 一个 gvfs 位置，位于“文件” |
| Windows | [go-mswin/win32](https://github.com/go-mswin/win32)（`MSFT_SmbMapping.Create`，任意 TCP 端口，标准用户） | 一个驱动器号，位于“此电脑” |

任何其他 GOOS 都能编译，并返回 `automount.ErrUnsupported`。

```go
u, _ := url.Parse("smb://127.0.0.1:4455/theca")
m, err := automount.Mount(ctx, automount.Share{URL: u, User: "alice", Password: pw}, automount.Options{})
if err != nil { /* errors.Is(err, netfs.ErrAuth) etc. */ }
fmt.Println(m.Path())   // /Volumes/theca, Z:\, /run/user/1000/gvfs/...
defer m.Unmount(ctx)
```

`Mount` 在接触操作系统之前就会拒绝：空的或非 `smb` 的 URL、缺少主机或共享、
带子路径的共享、错误的端口、URL 中的用户信息、无效或已占用的 Windows 驱动器，
以及无法满足的 `ReadOnly`（仅 macOS 支持）。密码绝不会进入参数向量、环境变量、
URL、错误消息或日志。

## 实测情况与地点 {#measured}

- **macOS 27.2.1（arm64）：已实测。** [go-filesystems/smb](https://github.com/go-filesystems/smb)
  的 SMB 服务器在测试进程内运行于 `127.0.0.1:<随机端口>`；门面挂载它，通过
  `Path()` 读写文件，`Unmount` 移除该卷。错误的密码以 `netfs.ErrAuth` 失败。
- **Linux（Ubuntu 24.04，gvfs 1.54.4）：已测量。** 在无显示的会话中
  （`dbus-run-session`、gvfsd、gvfsd-smb、gvfsd-fuse）执行与 macOS 相同的步骤，
  错误的密码得到 `gvfs.ErrAuthFailed`。未测量：真实的 GNOME 桌面会话。该次运行
  发现 go-freedesktop/gvfs v0.1.0 的一个缺陷（gvfs-fuse 未服务该会话时返回 `/`
  作为路径），已在 v0.1.1 修复；此时 `Path()` 是 `smb://` URI，GIO 能打开，
  `os` 调用不能。
- **Windows 11 25H2（arm64，QEMU）：已以标准用户和管理员身份测量**
  （v0.1.2 增加了标准用户的运行）：错误的密码得到 `*win32.SMBPortError`，映射 `Z:`，
  经它读写，卸载后 `Z:` 消失。未测量：资源管理器“此电脑”中盘符的出现
  （ssh 登录有自己的登录会话，所以检查通过 `os` 调用进行）。

[theca](https://github.com/go-theca/theca) 的 `serve --mount` 使用它。
