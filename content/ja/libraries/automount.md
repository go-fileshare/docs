---
title: "automount"
weight: 10
description: "パスワードダイアログなしで、SMB 共有を OS のファイルマネージャにマウントし、アンマウントする。"
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.2 は、
**パスワードダイアログを一切出さずに** SMB 共有を OS のファイルマネージャに
マウントし、アンマウントします。既存の 3 つのバックエンドの上に載せた、純粋な
Go（cgo なし）の小さなファサードです。このページは
[README](https://github.com/go-fileshare/automount/blob/main/README.md)
をもとに書かれています。リポジトリと異なる場合はリポジトリが優先されます。

| OS | バックエンド | 現れるもの |
|---|---|---|
| macOS | [go-macos/netfs](https://github.com/go-macos/netfs)（NetFS、`NoUI`） | `/Volumes` 下のボリューム（Finder のサイドバー） |
| Linux（GNOME） | [go-freedesktop/gvfs](https://github.com/go-freedesktop/gvfs)（GVfs 自身の `AskPassword` に応答） | gvfs の場所（ファイル） |
| Windows | [go-mswin/win32](https://github.com/go-mswin/win32)（`MSFT_SmbMapping.Create`、任意の TCP ポート、標準ユーザー） | ドライブ文字（「PC」） |

その他の GOOS はコンパイルでき、`automount.ErrUnsupported` を返します。

```go
u, _ := url.Parse("smb://127.0.0.1:4455/theca")
m, err := automount.Mount(ctx, automount.Share{URL: u, User: "alice", Password: pw}, automount.Options{})
if err != nil { /* errors.Is(err, netfs.ErrAuth) etc. */ }
fmt.Println(m.Path())   // /Volumes/theca, Z:\, /run/user/1000/gvfs/...
defer m.Unmount(ctx)
```

`Mount` は OS に触れる前に、次のものを拒否します。nil または `smb` 以外の URL、
ホストまたは共有名の欠落、サブパスを持つ共有、不正なポート、URL 内のユーザー
情報、不正または使用中の Windows ドライブ、そして守れない場所での `ReadOnly`
（macOS のみ可能）。パスワードは、引数ベクタ、環境変数、URL、エラーメッセージ、
ログのいずれにも入りません。

## 計測と、その場所 {#measured}

- **macOS 27.2.1（arm64）：計測済み。** [go-filesystems/smb](https://github.com/go-filesystems/smb)
  の SMB サーバーがテストプロセス内の `127.0.0.1:<ランダムなポート>` で動き、
  ファサードがそれをマウントし、`Path()` 経由でファイルを読み書きし、
  `Unmount` がボリュームを取り除きます。誤ったパスワードは `netfs.ErrAuth` で失敗します。
- **Linux（Ubuntu 24.04、gvfs 1.54.4）：計測済み。** 画面のないセッション
  （`dbus-run-session`、gvfsd、gvfsd-smb、gvfsd-fuse）で macOS と同じ手順を実行し、
  誤ったパスワードは `gvfs.ErrAuthFailed` になります。未計測：実際の GNOME
  デスクトップセッション。この実行で go-freedesktop/gvfs v0.1.0 の不具合
  （gvfs-fuse がセッションを提供していないとき `/` がパスとして返る）が見つかり、
  v0.1.1 で修正されました。その場合 `Path()` は `smb://` URI で、GIO は開けますが
  `os` の呼び出しでは開けません。
- **Windows 11 25H2（arm64、QEMU）：標準ユーザーと管理者で計測済み**
  （v0.1.2 で標準ユーザーでの実行が加わりました）。誤ったパスワードは
  `*win32.SMBPortError` になり、`Z:` を割り当て、経由して読み書きし、アンマウントすると
  `Z:` が消えます。未計測：エクスプローラーの「PC」に文字が現れること
  （ssh ログオンは独自のログオンセッションを持つため、確認は `os` の呼び出しで行います）。

[theca](https://github.com/go-theca/theca) の `serve --mount` が使っています。
