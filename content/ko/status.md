---
title: "현황"
weight: 60
description: "이 페이지들이 설명하는 fileshare 릴리스, 각 릴리스가 추가한 것, 실제 클라이언트로 검증된 것, 그리고 아직 구현되지 않은 것."
tags: [상태, 릴리스]
---

이 페이지들은 **fileshare v0.31.0**을 설명합니다.

## 각 릴리스가 추가한 것 {#what-each-release-added}

| | |
|---|---|
| v0.7.0 | [`kerberos` 블록]({{< relref "/protocols/nfs.md#the-refusal-that-used-to-be-permanent" >}})을 통한 NFS의 제한된 공유 |
| v0.8.0 | gRPC 기반 [관리 API]({{< relref "/administration/_index.md" >}}), [디렉터리 공유]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}), [`/healthz`, `/readyz`, `/metrics`]({{< relref "/administration/health.md" >}}) |
| v0.9.0 | 파일 또는 ACME의 인증서를 쓰는 WebDAV, S3, NFS용 [TLS]({{< relref "/security/tls.md" >}}), **비밀번호를 쓰는 WebDAV의 평문 거부** |
| v0.10.0 | [디렉터리 다시 읽기]({{< relref "/administration/reload.md" >}}): `reload = "5m"`, `SIGHUP`, `ReloadDirectory` |
| v0.11.0 | [ID 공급자가 보증하는 사용자를 위한 SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}): 공급자의 SSH CA와 opkssh |
| v0.12.0 | 폐기: 공급자의 SSH 인증서를 위한 [KRL]({{< relref "/security/revocation-lists.md" >}}), [인증서 기반 NFS ID]({{< relref "/security/nfs-certificates.md" >}})를 위한 CRL |
| v0.13.0 | [공유 신호]({{< relref "/security/shared-signals.md" >}}): CAEP `session-revoked`가 공급자가 그 이전에 발급한 모든 것을 모든 프로토콜에서 무효화 |
| v0.14.0 | 보안 검토의 수정 사항: [프로토콜이 담을 수 있는 공유 이름]({{< relref "/administration/_index.md#what-it-will-and-will-not-touch" >}}), 구성이나 그 비밀 정보를 포함하는 공유 금지, 시작할 때 `source_roots` 재검사, SSF `max_age` ≥ 1m 및 `retain` ≥ 169h |
| v0.15.0 | [`ssh_krl_url`의 KRL은 서명되고 날짜가 기록되어야 함]({{< relref "/security/revocation-lists.md#signed-and-dated-from-ssh_krl_url" >}}) |
| v0.16.0 | 폐기 목록이 [결코 거꾸로 가지 않음]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}}), `ssh_krl_state_file` / `crl_state_file`로 재시작 후에도 유지. 신뢰 앵커와 목록 파일은 공유 안에 있을 수 없음 |
| v0.16.1 | Go 1.26.6 필요: 1.26.4의 도달 가능한 표준 라이브러리 권고 9건 |
| v0.16.2 | [NFS CRL의 서명을 파싱하기 전에 검사]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}) |
| v0.16.3 | [검증되지 않은 이메일로는 누구도 지정되지 않음]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}) |
| v0.16.4 | 테스트만 |
| v0.16.5 | 테스트만([#36](https://github.com/go-fileshare/fileshare/issues/36): 테스트가 전송 오류를 성공으로 읽음) |
| v0.16.6 | NFS 거부 메시지가 두 가지 해결 방법을 모두 안내: `kerberos` 블록 또는 [`identity = "certificate"`]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.16.7 | go-authn/oidc v0.2.2: RSA 키의 크기를 비트 단위로 계산. 이전에는 0 옥텟으로 채운 1024비트 키가 WebDAV 토큰이나 opkssh PK 토큰을 검증했음 |
| v0.17.0 | 보안 감사의 수정 사항: 한 공유가 다른 공유를 포함할 수 없음, 접근 파일(`authorized_keys_file`, `dsn_file` 등)은 공유 안에 있을 수 없음, 디렉터리 읽기가 비어서 돌아와도 WebDAV가 익명이 되지 않음, S3가 `protocols`를 준수, 30초 안에 인증하지 않은 연결은 닫힘, opkssh 세션은 `opkssh_max_age`에 종료, 폐기된 세션은 폐기된 채로 닫힘 |
| v0.17.1 | go-authn/oidc v0.2.4(http로의 리디렉션 뒤에 키 집합 없음), servercert v0.3.0(sshd의 StrictModes처럼 ACME 캐시 경로를 검사), krl v0.5.0, revocation v0.3.0 |
| v0.17.2 | go-filesystems/s3 v0.3.0: S3가 인증되지 않은 호출자에게 어떤 액세스 키가 존재하는지 더 이상 알려 주지 않음. 사전 서명 URL이 동작하며, 최대 일주일간 유효하고, 자체 날짜 이전에는 거부됨 |
| v0.18.0 | CI가 `stable` 대신 Go 1.27.1을 고정 |
| v0.18.1 | `oidc` 블록에 `domains`가 없으면 `fileshare check`가 경고: [공급자의 단순 이름이 같은 이름의 로컬 계정에 도달]({{< relref "/configuration/identity.md" >}}) |
| v0.19.0 | go-authn/directory v0.11.0: `users "ldap"` 블록이 [다른 머신으로의 평문 바인드]({{< relref "/configuration/identity.md" >}})를 거부. 비밀번호는 다이제스트로 상수 시간에 비교됨 |
| v0.20.0 | [`local_names`]({{< relref "/configuration/identity.md" >}}): 공급자의 이름은 `oidc` 블록이 그렇다고 할 때만 로컬 이름(보안 감사 F4) |
| v0.21.0 | [볼륨]({{< relref "/administration/volumes.md" >}}): 권한 있는 `fileshare provisioner`가 ZFS 데이터셋, btrfs 서브볼륨, XFS/ext4 프로젝트 쿼터 디렉터리를 생성하고 관리 API가 그로부터 공유를 제공. 가득 찬 디렉터리 공유는 모든 프로토콜에서 하나의 응답. 관리 소켓의 `allowed_uids` |
| v0.21.1 | [가득 찬 공유]({{< relref "/administration/volumes.md#a-full-share" >}})가 SMB(`STATUS_DISK_FULL`)와 NFS(`NFS3ERR_NOSPC` / `NFS3ERR_DQUOT`)에서 그렇게 응답 |
| v0.22.0 | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}): SSH 인증서는 도메인 부여(`ssh-domain-grant@core.aai.geant.org`, EuroHPC SSH CA 프로필)가 이 호스트를 지정하는 곳에서만 수락되며 fail closed. `ssh_accept_ungranted`는 부여가 없는 인증서를 허용 |
| v0.22.1 | go-filesystems/sftp v0.5.1: [주소에 고정된]({{< relref "/protocols/sftp.md#a-certificate-pinned-to-an-address" >}}) 인증서(`source-address`)가 `ssh_domains` 아래와 공급자 및 OpenPubkey 인증서에서 그 주소로부터 수락됨. 이전에는 모든 주소에서 거부되었음 |
| v0.22.2 | 테스트와 문서만: `source-address`를 `::1`과 두 번째 IPv4 주소(`127.0.0.2`)에서, `ssh_domains` 아래에서, 그리고 OpenPubkey 인증서에서 테스트. Linux에서 `FILESHARE_REQUIRE_JUDGE`가 설정되면 판정기가 없을 때 건너뛰지 않고 실패 |
| v0.23.0 | [btrfs 볼륨]({{< relref "/administration/volumes.md#the-size-a-client-sees" >}})이 NFS, SMB, WebDAV에서 쿼터의 크기로 표시됨(go-filesystems/nfs v0.7.0, smb v0.5.0, webdav v0.3.0이 질의마다 크기를 보고). 이전에는 btrfs가 파일 시스템 전체를 보여 주었음. 공유가 관리 API에서 오는 서버가 NFS를 제공하면서 아직 공유가 없을 때 시작에서 더 이상 중단되지 않음 |
| v0.24.0 | 더 빠른 읽기: 디렉터리 또는 볼륨 공유의 WebDAV GET이 평문 HTTP에서 `sendfile(2)`로 전송됨(TLS에서는 아님), NFS 읽기와 쓰기는 1 MiB(rtmax/wtmax), SMB 클라이언트는 요청당 64 KiB보다 많이 읽음(`CAP_LARGE_MTU`). CI 러너에서 256 MiB 파일: WebDAV 6.6 GB/s, SMB 0.84, NFS 1.47(클라이언트 하나) |
| v0.24.1 | 보안: 유휴 NFS 연결이 가장 큰 호출에 필요했던 3 MiB를 더 이상 유지하지 않음(유휴 연결 24개가 59 MiB를 보유했음). SMB 프레임 헤더가 바이트가 도착하기 전에 헤더가 알린 8 MiB를 서버가 할당하게 하지 않음. SMB 요청이 크레딧 하나로 4 GiB를 지불할 수 없음(go-filesystems/nfs v0.8.1, smb v0.6.1 및 v0.6.2) |
| v0.25.0 | SFTP가 `limits@openssh.com`에 응답하여 OpenSSH의 `sftp`가 최대 255 KiB씩 읽음(+25%). 한 세션은 최대 1024개의 열린 핸들을 보유(go-filesystems/sftp v0.6.0) |
| v0.26.0 | 이를 지원하는 모든 프로토콜에서 서버 측 복사: WebDAV `COPY`, SFTP `copy-data`(OpenSSH의 `sftp cp`), SMB COPYCHUNK(탐색기, Linux `copy_file_range`)이 호스트 파일 간의 `copy_file_range(2)`로 서버에서 수행됨 — btrfs와 XFS에서는 리플링크이며, CI에서 8 MiB 복사에 사용량 0바이트로 측정됨. 보안: WebDAV `COPY`와 모든 S3 `GET`/`HEAD`가 더 이상 파일 전체를 메모리로 읽지 않음. S3는 모든 크기의 오브젝트를 스트리밍. 정정: v0.24.0의 WebDAV `GET`용 sendfile은 읽기 전용 공유에서만 일어났음 — 쓰기 가능한 공유에서는 래퍼가 이를 가렸고, v0.26.0부터 양쪽에서 일어남 |
| v0.27.0 | ACME: 인증서를 첫 연결이 아니라 **시작할 때** 요청하며, 서버가 계속 제공하는 동안 1분 후에 재시도하고 최대 1시간까지 두 배씩 늘림. **SNI를 보내지 않는** 클라이언트(IP 주소로 접속)에는 핸드셰이크 실패 대신 첫 번째 도메인의 인증서를 제공(go-authn/servercert v0.5.0). [HARICA]({{< relref "/security/tls.md#harica-gant-tcs-which-account-and-caa" >}}): Enterprise **Admin**(OV) 계정만 챌린지 없이 발급하며, CAA가 `harica.gr`을 허용해야 함 |
| v0.28.0 | 관리 API가 [OIDC 토큰을 위해 HTTPS로도]({{< relref "/administration/_index.md#over-https-for-oidc-tokens" >}}) 제공됨 — 하나의 리스너에서 Connect, gRPC-Web, gRPC, 서버마다 하나의 audience, 발급자별 주체 또는 그룹으로 지정된 호출자 — 웹 및 네이티브 UI를 향한 첫걸음 |
| v0.28.1 | 보안: Go 1.27.2와 golang.org/x/net v0.60.0으로 빌드하여 GO-2026-6617(서버를 중단시킬 수 있는 HTTP/2 HPACK 경쟁 상태), GO-2026-6613, -6612, -6611, -6610, -6603을 수정. HTTPS 위의 관리 API가 HTTP/2를 사용. `go install`은 이제 Go 1.27.2가 필요하며 스스로 전환함 |
| v0.29.1 | [다시 불러오기]({{< relref "/administration/reload.md" >}})가 디렉터리에서 비워진 그룹을 삭제된 그룹처럼 취급: 아무것도 부여하지 않으며, 다시 불러오기의 참고 사항에 `group @x has no members: it grants nothing`이 표시됨(빈 그룹과 없는 그룹을 구별하는 go-authn/directory v0.12.0) |
| v0.29.2 | **NetBSD(amd64) 및 OpenBSD arm64용 바이너리**: fileshare는 NetBSD에서 빌드되지 않았는데, memguard와 OpenPubkey를 통해 가져오는 awnumar/memcall v0.1.2가 NetBSD에 없는 플래그 `MADV_DONTDUMP`를 사용하기 때문입니다. memguard v0.23.0과 memcall v0.4.0이 업스트림에서 수정했습니다. 이제 모든 릴리스 플랫폼이 모든 풀 리퀘스트에서 크로스 빌드됩니다. v0.29.1부터 모든 릴리스는 [릴리스 페이지](https://github.com/go-fileshare/fileshare/releases)에 플랫폼별 정적 바이너리를 `SHA256SUMS` 및 출처 증명과 함께 제공합니다 |
| v0.29.0 | WebDAV: [루트의 `PROPFIND`가 공유를 나열]({{< relref "/protocols/webdav.md#the-root-and-uploads-in-pieces" >}})하여(207, Depth infinity는 403, 쓰기는 405) 파일 관리자가 서버 전체를 마운트할 수 있음. 업로드를 조각으로 보낼 수 있으며 각 `Content-Range`는 직전 조각이 끝난 곳에 이어짐(go-filesystems/webdav v0.6.0) — [파일 탐색기]({{< relref "/administration/_index.md" >}})가 필요로 하는 것 |
| v0.30.0 | go-authn/directory v0.14.0: [`users "ldap"` 블록]({{< relref "/configuration/identity.md" >}})이 기본적으로 go-authn/authnd v0.11.0이 게시하는 것을 읽음 — 사용자 `(\|(objectClass=posixAccount)(objectClass=account)(objectClass=uidObject))`, 그룹 `(\|(objectClass=posixGroup)(objectClass=groupOfNames))`, `member` DN은 사용자 검색이 반환한 사용자의 DN일 때만 인정. ⛔ 기관 자체의 디렉터리에서는 기본적으로 더 많은 항목을 읽음. 명시적인 `user_filter` 또는 `group_filter`는 변하지 않음 |
| v0.31.0 | [각 사용자의 권한으로 제공되는 디렉터리]({{< relref "/configuration/as-each-person.md" >}}): `identity = "system"`과 `identity` 블록. root로 시작한 `fileshare serve`가 사용자와 공유마다 워커를 시작하여 해당 uid, gid, 그룹으로 실행하고, 프로토콜은 `service_user`로 제공. SFTP와 WebDAV만 해당. uid 0, `min_uid` 미만의 uid, `deny_uids`는 거부됨 |

{{< callout type="warning" >}}
**v0.21.0으로 업그레이드할 때**

**가득 찬 디렉터리 공유**의 응답이 달라집니다. `EDQUOT`(쿼터)와 `ENOSPC`가 이제
볼륨만이 아니라 모든 디렉터리 공유에서 하나의 오류, "no space left on device (the
share is full)"입니다. WebDAV `500`이고 NFS `NFS3ERR_IO`였던 `EDQUOT`는 이제 `507`과
`NFS3ERR_NOSPC`이며, SFTP의 텍스트는 경로 없이 그 문장입니다. SMB는 여전히 접근
거부로 응답합니다. [가득 찬 공유]({{< relref "/administration/volumes.md#a-full-share" >}})를
참고하십시오.
{{< /callout >}}

{{< callout type="warning" >}}
**v0.20.0으로 업그레이드할 때**

공급자의 `alice`는 더 이상 로컬 `alice`의 공유를 받지 못합니다. 공급자의 이름이
곧 로컬 이름인 사이트는 `oidc` 블록에 `local_names = true`를 추가하며(그리고
`domains`를 설정하며), 서버는 이것이 없어 거부한 이름마다 해당 행을 로그에
기록합니다.
{{< /callout >}}

{{< callout type="warning" >}}
**v0.19.0으로 업그레이드할 때**

`start_tls = true` 없이 다른 머신의 `ldap://`를 가리키는 `users "ldap"` 블록은
**더 이상 시작되지 않습니다**. `ldaps://` 또는 StartTLS를 사용하십시오.
{{< /callout >}}

{{< callout type="warning" >}}
**v0.17.0으로 업그레이드할 때**

한 공유가 다른 공유 안에 있거나, 두 공유가 하나의 소스를 쓰거나, 접근 파일
(`authorized_keys_file`, `dsn_file`, `bind_password_file`, `ssf`의 `ca_file`)이
공유 안에 있는 구성은 **더 이상 시작되지 않으며**, 오류가 공유와 경로를 알려 줍니다.
{{< /callout >}}

{{< callout type="warning" >}}
**v0.9.0 이후로 업그레이드할 때**

인증할 사용자가 있는 상태에서 다른 머신이 접근할 수 있는 주소로 TLS 없이 WebDAV를
제공하는 구성은 **더 이상 시작되지 않습니다**. `tls = true`와 `tls` 블록을
추가하거나, TLS를 종료하는 프록시 뒤에서는 `plaintext = true`를 추가하십시오.
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})를 참고하십시오.
{{< /callout >}}

## 검증됨 {#verified}

이 프로젝트가 작성하지 않은 클라이언트가 실제로 수행하도록 한 것들이며, 코드에
대한 주장이 아닙니다.

- **macOS**는 `curl`이 같은 이미지에 WebDAV로 쓰는 동안 공유를 SMB로 마운트하고,
  WebDAV가 쓴 내용을 다시 읽습니다. 읽기만 가능한 공유에 대한 Bob의 WebDAV 쓰기는
  `403`이고, 잘못된 비밀번호는 `401`입니다.
- **go-smb2** — 이 프로젝트가 작성하지 않은 클라이언트 — 가 CI에서 `-race`로 SMB를
  통한 동시 읽기 스무 건과 WebDAV를 통한 스무 건을 함께 구동합니다.
- 접근 규칙은 구성만이 아니라 **모든 프로토콜을 통해** 검사됩니다. alice는 쓸 수
  있고 bob은 쓸 수 없는 것이 SMB와 WebDAV 양쪽에서 확인되며, NFS에는 그 공유가
  아예 제공되지 않습니다.
- SFTP 인증서는 **OpenSSH 자체 클라이언트**로 검증되며, 이 클라이언트는 인증서를
  치워 두면 같은 키를 거부합니다.
- 인증서 기반 NFS ID는 go-filesystems/nfs의 CI에서 **실제 Linux 클라이언트**(커널
  6.17, ktls-utils 0.9)로 측정했으며, 이를 통해
  [평문 `MNT`와 `tlshd`의 함정]({{< relref "/security/nfs-certificates.md#measured-with-a-real-linux-client" >}})을
  발견했습니다.
- KRL의 서명되지 않은 형태는 측정 결과입니다. OpenSSH 9.6과 10.3은 서명된 KRL을
  읽고 서명을 건너뜁니다(go-authn/krl, 두 버전 모두 대조).
- **go-authn/bridge의 상호 운용 레인**
  ([#12](https://github.com/go-authn/bridge/pull/12),
  [#17](https://github.com/go-authn/bridge/pull/17))이 fileshare v0.13.0을 실제
  bridge에 대해 판정합니다. KRL, NFS의 CRL, 공유 신호로 폐기된 WebDAV 토큰, 스코프로
  비활성화된 IdP, 공유 신호를 통한 opkssh입니다.

## 아직 지원하지 않는 것 {#not-yet}

**S3를 통한 쓰기.** `PUT`과 `DELETE`는 403으로 응답합니다. 라이브러리는 쓸 수
있지만, 사용자가 읽기만 할 수 있는 공유는 SFTP와 같은 지점에서 거부해야 하는데
아직 연결되지 않았습니다. 거부하는 편이 반쯤 쓰인 오브젝트보다 낫습니다.

**S3를 통한 OIDC 토큰.** 토큰은 [WebDAV]({{< relref "/protocols/webdav.md" >}})에서만
동작하며 다른 곳에서는 동작하지 않습니다. S3의 일반적인 경로는 STS
`AssumeRoleWithWebIdentity`로, 토큰을 클라이언트가 이후 서명에 쓰는 임시 자격
증명으로 교환합니다. 베어러 토큰을 수락하는 것과는 다른 메커니즘이며 구현되어 있지
않습니다.

**`admin` 또는 `metrics` 블록이 있는 `--isolate`.** 거부됩니다. API 변경을 적용할
단일 프로세스가 없기 때문입니다.
[프로토콜마다 하나의 프로세스]({{< relref "/operations/isolation.md#not-with-an-admin-or-a-metrics-block-yet" >}})를
참고하십시오.

**공유 Linux NFS 클라이언트에서 인증서 기반의 사용자별 구분.** Linux 클라이언트는
인증서를 마운트에 연결하므로 마운트의 모든 사용자가 인증서가 지정하는 사용자가
됩니다. 이는 클라이언트의 속성이며, 그곳에서의 답은 `sec=krb5`입니다.
[보장할 수 없는 것]({{< relref "/security/nfs-certificates.md#what-it-cannot-promise" >}})을
참고하십시오.

[S3 자체]({{< relref "/protocols/s3.md" >}})는 출시되었습니다. 공유는 버킷이며, SFTP가
사용하는 것과 같은 사용자별 트리 위에서 제공됩니다.

## 측정되었으며, 측정된 것으로 서술합니다 {#measured-and-stated-as-measured}

이 페이지들의 주장 중 결함의 재현이 아니라 보험이나 측정인 것들이며, 소스에도
그렇게 기록되어 있습니다.

- 감싸지 않은 fat32 드라이버를 `-race`로 실행하며 고루틴 여덟 개가 쓰기를 해도 오늘
  경쟁 상태가 발생하지 않습니다. [공유 잠금]({{< relref "/operations/locking.md" >}})은
  계약에 대한 보험(`go-filesystems/interface`는 경로 기반 동시 호출에 대해 아무것도
  보장하지 않음)이지, 관찰된 실패의 수정이 아닙니다.
- [빌드 태그 크기]({{< relref "/operations/build-tags.md" >}})는 측정한 값입니다.
  `hashicorp/go-plugin` 단독으로 드는 13.2 MB(플러그인 프레임워크를 택하지 않기로
  결정한 근거)와, 관리 API가 가져오는 gRPC의 11.7 MB(`nogrpc`가 존재하는 이유)를
  포함합니다.
- 그룹 태그용 자체 인증서 확장을 설계하여 Go의 X.509 파서에 대해 **측정**했고, 파서가
  인증서 전체를 거부했습니다. 그래서 그룹은 대신
  [tag URI]({{< relref "/security/nfs-certificates.md#what-the-certificate-carries" >}})로
  전달됩니다.
