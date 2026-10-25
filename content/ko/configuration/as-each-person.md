---
title: "각 사용자의 권한으로 제공되는 디렉터리"
weight: 15
description: "identity = \"system\": 로그인한 사용자의 권한으로 실행되는 워커가 파일을 여는 디렉터리 공유로, 커널이 트리의 권한을 적용하고 사용자가 만든 것은 그 사용자의 소유가 됩니다."
tags: [구성, 공유, ID, 보안]
---

보통은 하나의 fileshare 프로세스가 모든 사람에게 하나의 계정으로 서비스합니다.
누가 무엇을 할 수 있는지는 fileshare가 결정하고, 생성하는 모든 파일은 그
계정의 소유가 됩니다. 디렉터리 공유는 대신 **각 사용자의 권한으로** 제공될 수
있습니다. 로그인한 사용자의 파일은 그 사용자의 uid, gid, 그룹으로 실행되는
프로세스가 열므로, 커널이 트리의 권한을 적용하고 사용자가 만든 것은 그 사용자의
소유가 됩니다.

```hcl
identity {
  service_user = "fileshare"   # the account the protocols are served as
  min_uid      = 1000          # the default; uid 0 is refused whatever this says
  min_gid      = 1000
  deny_uids    = [65534]
  idle         = "2m"          # how long an unused worker is kept
}

share "home" {
  directory = "/srv/home"
  identity  = "system"
}
```

## 세 개의 프로세스 {#three-processes}

`fileshare serve`는 반드시 **root로 시작**해야 합니다. 각 사용자의 워커를
시작하기 위해서만 root로 남아 있습니다. 네트워크와 맞닿는 모든 것은 두 번째
프로세스에서 `service_user`로 실행됩니다.

| 프로세스 | 실행 계정 | 역할 |
|---|---|---|
| **런처**(`fileshare serve`) | root | 리스너를 바인딩하고, 프런트를 시작하며, 프런트가 요청하면 워커를 시작합니다. 어떤 프로토콜도 사용하지 않습니다. |
| **프런트** | `service_user`와 그 계정의 그룹 | `serve`가 항상 해 왔듯이 프로토콜을 사용하고 인증합니다. 이런 공유에 대해서는 uid나 경로가 아니라 **사용자와 공유**를 런처에 요청합니다. |
| **워커** | 해당 사용자 | 런처가 넘겨준 소켓을 통해 프런트에 공유 하나를 제공합니다 |

워커는 먼저 자신이 정확히 기대된 uid, gid, 그룹으로 실행 중인지, 그리고
`setuid(0)`이 실패하는지 확인합니다. 그런 다음에야 `os.Root`를 통해 트리를
엽니다. 워커 하나가 SFTP와 WebDAV를 가리지 않고 그 사용자의 해당 공유에 대한
모든 연결을 처리하며, `idle` 동안 아무도 사용하지 않으면 종료됩니다. 서버가
중지되면 워커도 함께 중지됩니다.

## 이 시스템에서 사용자가 누구인가 {#who-a-person-is-on-this-system}

런처는 사용자를 **직접** 다음 순서로 확인합니다.

1. **디렉터리**에 `uidNumber`와 `gidNumber`가 있는 경우:
   - `user` 블록의 `attributes = { uidNumber = "2001", gidNumber = "2001" }`
   - `attributes = ["uidNumber", "gidNumber"]`가 있는 LDAP 소스
   - SQL 소스의 `users_attributes` 쿼리

   보조 그룹은 해당 사용자를 나열하는 그룹(`group` 블록의 `members`,
   `memberUid`)과 해당 사용자가 이름으로 지정한 그룹(`memberOf`)이며, 각각
   `gidNumber`로 식별합니다. `gidNumber`가 없는 그룹은 시스템 그룹이 아닙니다.
2. 이름으로 찾는 **`/etc/passwd`와 `/etc/group`**.

**거부되는 경우**: uid 0, gid 0, `(uid_t)-1`, `min_uid` 미만의 uid, `min_gid`
미만의 기본 gid, `deny_uids`에 있는 모든 uid. `min_gid` 미만의 보조 그룹은
제외되며 로그에 그렇게 기록됩니다. 두 곳 어디에서도 찾을 수 없는 사용자도
거부됩니다. 공유 대체 계정은 없습니다. ID 공급자가 보증하는 사용자는
[`local_names`]({{< relref "/configuration/identity.md" >}})가 있어야만 워커를 받으며,
이 옵션이 그들의 이름을 로컬 이름으로 만듭니다.

따라서 탈취된 프런트가 요청할 수 있는 것은 존재하고 허용된 사용자와 존재하는
공유뿐입니다. **누가** 인증했는지는 여전히 프런트가 결정하며, 런처는 이를
대신 확인할 수 없습니다.

## 클라이언트에 보이는 것 {#what-a-client-sees}

| | SFTP | WebDAV |
|---|---|---|
| 워커를 받는 시점 | 로그인할 때, 연결 전체에 대해 | 요청마다. 잠금은 워커가 살아 있는 동안 유지됩니다 |
| 런처가 사용자를 거부함 | 공유가 그 사용자의 트리에 없음(아무것도 없는 트리는 거부된 로그인) | `403` |
| 워커를 시작할 수 없음 | 동일 | `503` |

이유는 서버의 로그에 기록되며 클라이언트에는 전달되지 않습니다.

{{< callout type="warning" >}}
**SMB, NFS, S3에서는 아직 지원하지 않습니다**

이들은 이런 공유를 거부하며, [`check`]({{< relref "/configuration/check.md" >}})가 이를
알려 줍니다. SMB와 NFS는 라이브러리에서 세션마다 파일 시스템이 필요합니다. NFS는
`AUTH_UNIX`가 클라이언트 스스로의 주장이므로(RFC 5531 §14) Kerberos 아래에서만
이를 지원하게 될 것입니다.
{{< /callout >}}

## 시스템에 요구하는 것 {#what-it-asks-of-the-system}

- **Unix**: Linux, BSD 계열, macOS. Windows에서는 공유가 거부됩니다.
- **`service_user`가 읽을 수 있어야 함**: 프런트가 읽으므로 구성과 구성이
  지정하는 모든 파일(비밀번호 파일, 호스트 키, 인증서).
- **사용자가 실행할 수 있어야 함**: 각 워커가 해당 사용자로 바이너리를
  실행하므로 바이너리.
- 이런 공유를 추가하거나 변경하려면 **재시작**이 필요합니다. 런처는 구성을 한 번만
  읽으며, 알지 못하는 공유는 프런트가 제공하게 두지 않고 거부합니다.
- `--isolate`와는 함께 쓸 수 없습니다.

설계와 각 선택의 이유는
[docs/identity.md](https://github.com/go-fileshare/fileshare/blob/main/docs/identity.md)에
있습니다.
