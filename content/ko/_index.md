---
title: "go-fileshare"
linkTitle: "홈"
type: docs
cascade:
  type: docs
description: "go-fileshare는 디스크 이미지와 디렉터리를 SMB, NFS, WebDAV, SFTP, S3로 제공하며, 하나의 구성에서 하나의 사용자 집합과 공유별 규칙을 적용합니다."
---

**SMB, NFS, WebDAV, SFTP, S3로 제공되는 디스크 이미지 또는 디렉터리 —
동일한 사용자, 동일한 공유별 접근 권한을 하나의 구성 파일에서 관리합니다.**
순수 Go, `CGO_ENABLED=0`, 바이너리 하나입니다. 이 페이지들은
[v0.32.0]({{< relref "/status.md" >}})을 설명합니다.

```sh
go install github.com/go-fileshare/fileshare@latest          # with Go 1.27.3 or later

fileshare --image disk.img --user alice --password-file pw   # one image, now
fileshare --config /etc/fileshare.d                          # several, with users
```

또는 **Go 없이 바이너리 하나로**: v0.29.1 이후의 모든 릴리스는
[릴리스 페이지](https://github.com/go-fileshare/fileshare/releases)에 플랫폼별
정적 바이너리를 제공합니다(Linux는 여섯 가지 아키텍처, macOS, Windows, FreeBSD,
OpenBSD, 그리고 v0.29.2부터 NetBSD). `sha256sum -c --ignore-missing SHA256SUMS`로
확인하고, `gh attestation verify <아카이브> -R go-fileshare/fileshare`로 GitHub가
태그에서 빌드했는지 확인하십시오.

비밀번호는 플래그가 아니라 **파일**에서 가져옵니다. 인자는 프로세스 목록에서 그
머신의 모든 사용자에게 보이기 때문입니다.

## 프로그램이 하나인 이유 {#why-one-program}

이미지를 어떤 프로토콜로 전달할지는 반대편 클라이언트의 속성입니다. macOS와
Windows는 SMB를, Linux 플릿은 이미 NFS를, 브라우저나 휴대폰은 HTTP를 사용합니다.
서버 세 개를 각자의 구성 파일과 `alice`가 누구인지에 대한 각자의 생각으로 운영하면,
그중 둘은 미묘하게 잘못 설정되기 쉽습니다.

그래서 구성도, 사용자 집합도, 공유별 규칙도 하나이며, 프로토콜은 그 위의
리스너입니다.

## 시작할 때 출력하는 것 {#what-it-prints-at-startup}

```
smb    on 0.0.0.0:445 — photos and scratch
webdav on 0.0.0.0:8080 — photos and scratch
sftp   on 0.0.0.0:2222 — photos and scratch
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

마지막 문단이 프로그램 전체의 성격을 보여 줍니다. 구성이 요구하는 규칙과 그것을
지킬 수 없는 프로토콜이 조용히 중간에서 타협하는 일은 없습니다.
[프로토콜이 보장할 수 있는 것]({{< relref "/protocols/_index.md" >}})을 참고하십시오.

## 다음에 볼 곳 {#where-to-go-next}

| | |
|---|---|
| [구성]({{< relref "/configuration/_index.md" >}}) | 사용자, 그룹, 공유, `serve` 블록 |
| [공유: 이미지, 장치, 디렉터리]({{< relref "/configuration/shares.md" >}}) | 무엇이 탐지되고 무엇을 선언해야 하는가 |
| [사용자, 그룹, 디렉터리]({{< relref "/configuration/identity.md" >}}) | 파일, SQL, LDAP, 그리고 각각이 증명할 수 있는 것 |
| [`check`]({{< relref "/configuration/check.md" >}}) | 재시작하기 전에 전체 구성을 다시 읽어 확인 |
| [프로토콜]({{< relref "/protocols/_index.md" >}}) | 각 프로토콜이 보장할 수 있는 것과 없는 것 |
| [관리 API]({{< relref "/administration/_index.md" >}}) | 재시작 없이 공유를 생성하고, 권한을 부여하고, 오프라인으로 전환 |
| [볼륨]({{< relref "/administration/volumes.md" >}}) | API로 생성하여 제공하는 ZFS 데이터셋, btrfs 서브볼륨, XFS/ext4 프로젝트 쿼터 |
| [상태 점검과 메트릭]({{< relref "/administration/health.md" >}}) | `/healthz`, `/readyz`, `/metrics` |
| [디렉터리 다시 읽기]({{< relref "/administration/reload.md" >}}) | `reload`, `SIGHUP`, 그리고 제거가 열린 세션에 미치는 영향 |
| [TLS]({{< relref "/security/tls.md" >}}) | 파일 또는 ACME, 그리고 WebDAV의 평문이 거부되는 이유 |
| [무엇이 무엇을 폐기하는가]({{< relref "/security/_index.md" >}}) | KRL, CRL, 공유 신호 — 그리고 각각이 안전하게 차단되는(fail closed) 이유 |
| [필요한 것만 빌드하기]({{< relref "/operations/build-tags.md" >}}) | 태그로 프로토콜을 완전히 제외 |
| [프로토콜마다 하나의 프로세스]({{< relref "/operations/isolation.md" >}}) | `--isolate` |
| [현황]({{< relref "/status.md" >}}) | 검증된 것과 아직 없는 것 |

## 라이선스 {#licence}

BSD-3-Clause.
