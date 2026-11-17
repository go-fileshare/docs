---
title: "automount"
weight: 10
description: "비밀번호 대화 상자 없이 SMB 공유를 운영 체제의 파일 관리자에 마운트하고 언마운트합니다."
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.2은
**비밀번호 대화 상자 없이** SMB 공유를 운영 체제의 파일 관리자에 마운트하고
언마운트합니다. 기존 백엔드 세 가지 위에 얹은 순수 Go(cgo 없음)의 작은
파사드입니다. 이 페이지는
[README](https://github.com/go-fileshare/automount/blob/main/README.md)를
바탕으로 작성되었으며, 저장소와 다르면 저장소가 우선합니다.

| OS | 백엔드 | 나타나는 것 |
|---|---|---|
| macOS | [go-macos/netfs](https://github.com/go-macos/netfs) (NetFS, `NoUI`) | `/Volumes` 아래의 볼륨, Finder 사이드바 |
| Linux (GNOME) | [go-freedesktop/gvfs](https://github.com/go-freedesktop/gvfs) (GVfs 자체의 `AskPassword`에 응답) | gvfs 위치, 파일 앱 |
| Windows | [go-mswin/win32](https://github.com/go-mswin/win32) (`MSFT_SmbMapping.Create`, 모든 TCP 포트, 표준 사용자) | 드라이브 문자, "내 PC" |

그 밖의 GOOS는 컴파일되며 `automount.ErrUnsupported`로 응답합니다.

```go
u, _ := url.Parse("smb://127.0.0.1:4455/theca")
m, err := automount.Mount(ctx, automount.Share{URL: u, User: "alice", Password: pw}, automount.Options{})
if err != nil { /* errors.Is(err, netfs.ErrAuth) etc. */ }
fmt.Println(m.Path())   // /Volumes/theca, Z:\, /run/user/1000/gvfs/...
defer m.Unmount(ctx)
```

`Mount`는 운영 체제를 건드리기 전에 다음을 거부합니다. nil 또는 `smb`가 아닌
URL, 호스트나 공유 이름 누락, 하위 경로가 있는 공유, 잘못된 포트, URL 안의
사용자 정보, 잘못되었거나 사용 중인 Windows 드라이브, 지킬 수 없는 곳에서의
`ReadOnly`(macOS만 가능). 비밀번호는 인수 벡터, 환경 변수, URL, 오류 메시지,
로그 어디에도 들어가지 않습니다.

## 측정한 것과 그 장소 {#measured}

- **macOS 27.2.1 (arm64): 측정함.** [go-filesystems/smb](https://github.com/go-filesystems/smb)의
  SMB 서버가 테스트 프로세스 안에서 `127.0.0.1:<임의 포트>`로 실행되고, 파사드가
  이를 마운트하며, `Path()`로 파일을 읽고 쓰고, `Unmount`가 볼륨을 제거합니다.
  잘못된 비밀번호는 `netfs.ErrAuth`로 실패합니다.
- **Linux (Ubuntu 24.04, gvfs 1.54.4): 측정함.** 화면 없는 세션
  (`dbus-run-session`, gvfsd, gvfsd-smb, gvfsd-fuse)에서 macOS와 같은 순서로
  실행했고, 잘못된 비밀번호는 `gvfs.ErrAuthFailed`가 됩니다. 측정하지 않음: 실제
  GNOME 데스크톱 세션. 이 실행에서 go-freedesktop/gvfs v0.1.0의 결함(gvfs-fuse가
  세션을 제공하지 않을 때 경로로 `/`가 반환됨)을 발견했고 v0.1.1에서 고쳤습니다.
  그 경우 `Path()`는 `smb://` URI이며 GIO는 열 수 있지만 `os` 호출은 열 수 없습니다.
- **Windows 11 25H2 (arm64, QEMU): 표준 사용자와 관리자로 측정함**
  (v0.1.2가 표준 사용자 실행을 추가했습니다). 잘못된 비밀번호는
  `*win32.SMBPortError`가 되고, `Z:`를 연결해 그것을 통해 읽고 쓰고, 언마운트하면
  `Z:`가 사라집니다. 측정하지 않음: 탐색기의 "내 PC"에 드라이브 문자가 나타나는
  것(ssh 로그온은 자체 로그온 세션을 가지므로 확인은 `os` 호출로 합니다).

[theca](https://github.com/go-theca/theca)의 `serve --mount`가 사용합니다.
