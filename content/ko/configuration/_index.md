---
title: "구성"
linkTitle: "구성"
weight: 10
description: "HCL 구성이 어떻게 배치되는지, user 블록이 담는 것, 그리고 시작할 때 거부되는 것."
tags: [구성]
---

파일 하나, 또는 작은 파일들이 모인 디렉터리입니다. `--config /etc/fileshare.d`는
이를 병합하므로, 한 파일의 사용자와 다른 파일의 공유가 함께 동작합니다.

```hcl
name = "ATTIC"

user "alice" { password_file = "/etc/fileshare/alice.pw" }
user "bob"   { password_file = "/etc/fileshare/bob.pw" }

group "family" { members = ["alice", "bob"] }

share "photos" {
  image   = "/srv/photos.img"
  allow   = ["@family"]        # a group, or a person, in either list
  writers = ["alice"]          # bob gets it read-only
}

share "scratch" {
  image = "/srv/scratch.img"   # anyone who authenticates, read-write
}

tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true                  # HTTP Basic is the password: never in the clear
}
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

{{< callout type="error" >}}
**v0.9.0부터 비밀번호를 쓰는 WebDAV는 평문으로 제공되지 않습니다**

그때까지 이 페이지에 실려 있던 예제인 `serve "webdav" { addr = "0.0.0.0:8080" }`는
이제 누군가 인증하는 경우 **시작할 때 거부됩니다**. HTTP Basic은 모든 요청에서
비밀번호 그 자체이기 때문입니다. 위와 같이 `tls` 블록과 함께 `tls = true`를
지정하거나, 프록시가 앞단에서 TLS를 종료하는 경우에는 `plaintext = true`를
지정하십시오. [TLS]({{< relref "/security/tls.md" >}})를 참고하십시오.
{{< /callout >}}

공유는 이미지 대신 [장치]({{< relref "/configuration/shares.md#a-device-not-only-an-image" >}})나
[호스트의 디렉터리]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})일
수도 있습니다.

## `user` 블록이 담을 수 있는 것 {#what-a-user-block-can-carry}

위의 모든 예제는 `password_file`을 사용합니다. 가장 흔한 경우이며 한동안은
이 프로그램이 읽는 유일한 필드였습니다. 하지만 블록의 전부는 아니며, 그 차이가
**어떤 프로토콜로 사용자에게 제공할 수 있는지**를 결정합니다. 마운트할 때
알게 되기보다 한곳에 정리해 두는 편이 좋습니다.

| 필드 | 설명 | 얻는 것 |
|---|---|---|
| `password` | 파일에 직접 적은 비밀번호 | 비밀번호로 응답하는 모든 프로토콜 |
| `password_file` | 같은 것을 별도 파일에 둠 | 같지만, 구성에 비밀 정보가 들어가지 않음 |
| `nt_hash` | `MD4(UTF16LE(password))`, 16진수 32자 | 이 사이트가 비밀번호를 보유하지 않는 사용자를 위한 **SMB** |
| `authorized_keys` | 인라인 `authorized_keys` 행 | **SFTP** |
| `authorized_keys_file` | 같은 것을 파일에서 읽음 | **SFTP** |
| `totp_secret` | base32 일회용 코드 비밀 | 아직 없음: 읽고 검사하지만, 여기의 어떤 프로토콜도 2단계 인증을 요구하지 않습니다 |

⛔ `password`와 `password_file`을 함께 쓰면 해당 사용자를 지목하며 시작할 때
거부됩니다. "비밀번호가 무엇인가"에 대한 답이 둘이면 어느 쪽이 우선하는지가
문제가 되고, 구성을 두 번 읽어야 알 수 있어서는 안 되기 때문입니다.
`authorized_keys`와 `authorized_keys_file`을 함께 쓰는 경우도 마찬가지입니다.

⛔ `password`, `password_file`, `authorized_keys`, `authorized_keys_file` 중
어느 것도 없는 `user` 블록도 *인증 방법이 없다*는 이유로 거부됩니다.
`trusted_user_ca_file`이 설정된 경우는 예외입니다. `nt_hash`와 `totp_secret`은 여기서
인정되지 않습니다.

{{< callout type="default" >}}
**인라인 사용자도 SMB로 제공할 수 있습니다**

얼마 전까지 이 블록은 비밀번호 필드만 읽었으므로, 여기에 적은 사용자에게
SMB를 제공하려면 이 파일에 비밀번호를 평문으로 적거나 그 사용자를
데이터베이스로 옮겨야 했습니다. `nt_hash`가 있으면 그럴 필요가 없습니다.
Samba가 저장하는 값을 저장하는 사이트라면 그 값을 대신 적을 수 있습니다. 블록에는
여전히 위 필드 중 하나(예: 키)나 `trusted_user_ca_file`이 함께 필요합니다.
`nt_hash`만 있으면 시작할 때 거부됩니다.
[소스가 증명할 수 있는 것]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}})을
참고하십시오. 한 계층 위에서 본 같은 표입니다.
{{< /callout >}}

그룹은 사용자가 올 수 있는 모든 곳에서 `@name`으로 씁니다.

## 나중이 아니라 시작할 때 거부되는 것 {#what-is-refused-at-startup-rather-than-later}

{{< callout type="warning" >}}
**누구의 것도 아닌 이름**

`allow = ["alise"]`는 그대로 두면 Alice를 자신의 공유에서 쫓아내고도 아무 일
없이 시작됩니다. 어떤 `user` 블록도, 어떤 [`users` 디렉터리]({{< relref "/configuration/identity.md" >}})도
모르는 이름은 거부되며, 그중 어느 곳에도 없는 `@group`도 마찬가지입니다.
{{< /callout >}}

{{< callout type="warning" >}}
**NFS에서, 사용 가능한 사용자를 지정한 공유**

*photos는 alice의 것*이라고 말하는 구성과 연결하는 누구에게나 photos를 내주는
프로토콜은 둘 다 충족될 수 없습니다. 이런 공유는 NFS로 내보내지 않습니다.
`kerberos` 블록이나 `identity = "certificate"`로 NFS가 사용자를 구별할 수 있는
경우는 예외입니다. 담을 것이 하나도 없게 된 `nfs` serve 블록도 거부됩니다.
[NFS]({{< relref "/protocols/nfs.md" >}})를 참고하십시오.
{{< /callout >}}

{{< callout type="warning" >}}
**자신의 말을 실을 곳이 없는 공급자**

[`oidc`]({{< relref "/protocols/webdav.md" >}}) 블록은 `Authorization` 헤더를 운반하는
유일한 프로토콜인 WebDAV가 제공되지 않거나, SFTP가 공급자의 인증서를 켠 상태
(`ssh_ca_file` 또는 `opkssh_client_id`,
[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}) 참고)로
제공되지 않는 한 거부됩니다. SMB와 NFS에는 둘 중 어느 것을 실을 곳도 없습니다.
{{< /callout >}}

{{< callout type="warning" >}}
**평문 비밀번호**

누군가 인증하는 상태에서, 다른 머신이 접근할 수 있는 주소의 WebDAV에
`tls = true`나 `plaintext = true`가 없는 경우입니다.
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})를 참고하십시오.
{{< /callout >}}

{{< callout type="warning" >}}
**안전하게 제공할 수 없는 블록**

`state_file`이 없거나 상호 TLS 없이 TCP에서 수신하는 `admin` 블록, 어떤 `serve`
블록도 사용하지 않는 `tls` 블록, 1초보다 짧은 `reload`, 평문 HTTP로 가져오는
폐기 목록입니다. 각각 해당하는 곳에 설명되어 있습니다.
[관리]({{< relref "/administration/_index.md" >}}),
[TLS]({{< relref "/security/tls.md" >}}), [다시 불러오기]({{< relref "/administration/reload.md" >}}),
[폐기]({{< relref "/security/_index.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**이 바이너리가 제외하고 빌드된 프로토콜**

"그런 프로토콜은 없다"가 아니라 *그렇게 빌드되었다*는 사실이 알려집니다.
오타와 [빌드 태그]({{< relref "/operations/build-tags.md" >}})의 차이입니다.
{{< /callout >}}

`addr`이 없는 `serve` 블록은 권한이 필요 없는 포트에서 루프백 `127.0.0.1`로
수신합니다. SMB는 4445, WebDAV는 8080, SFTP는 2222, NFS는 2049, S3는
9000입니다.

## 공유의 `protocols` {#protocols-on-a-share}

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

어떤 프로토콜이 이 공유를 제공하는지 지정합니다. 단독으로도 유용합니다.
SMB 전용으로 선언된 공유는 WebDAV 프로세스가 전혀 알지 못하는 공유이며,
[`--isolate`]({{< relref "/operations/isolation.md" >}}) 아래의 일부 구성에서는
필수입니다. 결국 아무것도 제공하지 않게 될 `serve` 블록도 이미지가 열리기 전에
거부되며, 그 블록에서 제외된 공유와 이유가 함께 표시됩니다. 나중에 공유를 줄
`admin` 블록이 있는 경우는 예외입니다.
