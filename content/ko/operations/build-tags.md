---
title: "필요한 것만 빌드하기"
weight: 10
description: "프로토콜, 사용자 디렉터리, 관리 API, 프로비저너를 바이너리에서 제외하는 빌드 태그와 각각이 줄여 주는 크기."
tags: [운영, 빌드]
---

각 프로토콜은 빌드 태그 뒤에 있으며, 태그는 그것을 **완전히** 제외합니다.
리스너도, 파서도, 의존성도, 코드도 없습니다.

```sh
go install -tags nonfs,nowebdav,nosftp,nos3 github.com/go-fileshare/fileshare@latest   # SMB only
go build   -tags nosmb,nonfs,nosftp,nos3 .                                            # WebDAV only
```

**사용자**가 어디에서 오는지도 자체 태그 뒤에 있습니다. `nosql`은 세 개의
데이터베이스 드라이버를, `noldap`은 LDAP 클라이언트를 제외합니다.

| 빌드 | 크기 |
|---|---|
| 전체 | 31.2 MB |
| `-tags noldap` | 30.9 MB |
| `-tags nosftp` | 30.6 MB |
| `-tags noopenpubkey` (SFTP를 통한 opkssh 로그인 없음) | 약 2 MB 감소 |
| `-tags nos3` | 31.1 MB |
| `-tags nonfs,nowebdav,nosftp,nos3` (SMB 전용) | 28.1 MB |
| `-tags nopartitioned` (apfs, btrfs, xfs, zfs 없음) | 29.2 MB |
| `-tags nosql` | 20.2 MB |
| `-tags nosql,noldap` | 19.8 MB |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp,nos3` | 16.2 MB |

이는 fileshare README가 제시하는 수치로, 관리 API가 추가되기 전에 측정한
것입니다. 아래 표가 그 이후의 측정입니다.

## 관리 API와 `nogrpc` {#the-admin-api-and-nogrpc}

[관리 API]({{< relref "/administration/_index.md" >}})는 gRPC와 protobuf를 끌어들였고,
이들은 `nogrpc` 뒤에 있습니다(linux/amd64, 2026-09-29 측정. 이때 위 표의 전체
빌드는 34.1 MB로 커져 있었습니다).

| 빌드 | 크기 |
|---|---|
| 전체, 관리 API 포함 | 46.1 MB |
| `-tags nogrpc` | 34.4 MB |
| `-tags nosql,noldap,nogrpc` | 22.5 MB |

gRPC의 비용은 **11.7 MB**입니다. 아래의
[플러그인 측정](#why-not-subprocess-plugins)을 이제 자체적인 이유로 다시 측정한
값입니다. 공유를 파일로 관리하는 사이트는 이를 싣고 있을 필요가 없습니다.

{{< callout type="warning" >}}
**`nogrpc` 빌드에서 `admin` 블록은 거부됩니다**

관리 API 없이 제공하는 대신 거부합니다. API를 요구하면서 API 없이 시작하는
구성은 말한 대로 동작하지 않기 때문입니다.
{{< /callout >}}

`nogrpc`는 관리 API와 함께
[프로비저너]({{< relref "/administration/volumes.md" >}})도 제외합니다.
[상태 점검과 메트릭]({{< relref "/administration/health.md" >}})은 평범한 HTTP이므로
유지됩니다.

`-tags noprovisioner`는 프로비저너, 곧 권한 있는 `fileshare provisioner` 역할만
제외하며, go-fsctl의 zfs, btrfs, projquota도 함께 제외됩니다. 볼륨을 전혀
제공하지 않는 서버는 볼륨을 만드는 코드를 싣고 있을 필요가 없습니다. 이런 빌드의
`fileshare provisioner`는 "알 수 없는 명령" 대신 그렇게 빌드되었음을 알려
줍니다.

`nosql`은 단연 가장 큰 수단입니다. PostgreSQL, MySQL, SQLite를 합치면
**11.7 MB**로, 이 프로그램의 모든 프로토콜을 합친 것보다 큽니다. 사용자가 파일에
있는 사이트에서는 이 태그가 필요합니다. 드라이버는 이를 사용하는 라이브러리가
아니라 명령이 임포트하며, 그래서 이 선택이 포크가 아닌 빌드 태그로 가능합니다.

{{< callout type="info" >}}
**이 바이너리가 제외하고 빌드된 프로토콜**

그런 프로토콜을 지정한 구성에는 "그런 프로토콜은 없다"가 아니라 *그렇게
빌드되었다*는 사실이 알려집니다. 오타와 빌드 태그의 차이입니다. `noopenpubkey`
아래의 `opkssh_client_id`와 `nogrpc` 아래의 `admin` 블록도 마찬가지입니다.
{{< /callout >}}

## WebAssembly 컨테이너로 (`wasip1`) {#as-a-webassembly-container-wasip1}

v0.32.0부터 fileshare는 `GOOS=wasip1 GOARCH=wasm`용으로 빌드되며 WebDAV를 WASI
컨테이너로 제공합니다. 예를 들어
[go-wasmcontainers](https://github.com/go-wasmcontainers) 엔진(`wctr`)에서
네이티브로 또는 브라우저 탭 안에서 실행할 수 있습니다.

```sh
GOOS=wasip1 GOARCH=wasm go build -trimpath \
  -tags nosql,noopenpubkey,nosmb,nonfs,nos3,nosftp,nogrpc,nopartitioned,noprovisioner \
  -o fileshare.wasm .                              # WebDAV 전용: gzip 6.5 MB
```

- `nosql`과 `noopenpubkey`는 필수입니다. SQLite 드라이버와 OpenPubkey의 의존성은
  wasip1용으로 빌드되지 않습니다. 전체 wasip1 빌드(`-tags nosql,noopenpubkey`)는
  gzip 11.5 MB입니다. CI에는 두 변형을 모두 빌드하는 `wasip1` 작업이 있습니다.
- **수신 대기**는 엔진의 소켓(`stealthrocket/net/wasip1`, WasmEdge 소켓 확장)을
  통해 이루어집니다. Go의 `net.Listen`은 wasip1에서 소켓이 없습니다.
- **시그널 처리 없음.** Go의 wasip1 런타임은 루프에서 양보하며 시그널을 기다리기
  때문에, `signal.Notify` 하나만으로도 네트워크 폴러가 굶어 리스너가 아무것도
  accept하지 못했습니다. 종료는 호스트가 모듈을 끝내는 방식입니다.
- **TLS**는 컨테이너 앞에서 끝납니다. 거기까지의 구간이 사설이면
  `plaintext = true`를 사용하세요.

{{< callout type="info" >}}
**go-wasmcontainers 플레이그라운드의 다음 릴리스에서 제공 예정**

`fileshare` 이미지와, go-fileshare 탐색기(v0.4.0)를 컨테이너의 브라우저
클라이언트로 여는 "explore :PORT" 버튼
([playground#31](https://github.com/go-wasmcontainers/playground/pull/31),
보류 중). 아직 사용할 수 없습니다.
{{< /callout >}}

## 서브프로세스 플러그인을 쓰지 않는 이유 {#why-not-subprocess-plugins}

논증하지 않고 **측정했습니다**. `hashicorp/go-plugin`은 gRPC와 protobuf를
끌어들이며, 이것만으로 **13.2 MB**가 듭니다. 세 프로토콜과 모든 드라이버를 담은
이 바이너리 전체보다 큽니다. 플러그인 호스트는 아무것도 로드하기 전에 이미 두 배
크기가 될 것이고, 각 플러그인 바이너리도 gRPC를 다시 싣게 됩니다.

공격 표면 측면에서도, 실행하지 않는 것에 대해서는 태그가 더 강력한 수단입니다.
**컴파일된 적이 없는 코드는** 샌드박스 여부와 관계없이 **도달할 수 없습니다**.

태그가 *주지 못하는* 것은 **실행하는** 프로토콜 사이의 격리, 그리고 다시
컴파일하지 않고 프로토콜을 추가하는 방법입니다. 둘 다 실제 요구이며, 둘 다 더
작은 바이너리가 아니라 프로세스 경계를 위한 논거입니다. 그 경계가 바로 플러그인
프레임워크가 아니라 같은 바이너리를 사용하는
[프로토콜마다 하나의 프로세스]({{< relref "/operations/isolation.md" >}})입니다.
