---
title: "관리 API"
linkTitle: "관리"
weight: 30
description: "서버가 실행 중인 동안 공유를 생성, 변경, 비활성화하고 권한을 부여하는 gRPC 관리 API, 이 API가 수신하는 위치, 그리고 거부하는 것."
tags: [관리, 관리 API, grpc]
---

v0.8.0부터 서버가 실행 중인 동안에도 공유를 바꿀 수 있습니다. `admin` 블록은
재시작이나 파일 편집 없이 공유를 생성하고 변경하고 오프라인으로 전환하는
gRPC 서비스를 켭니다.

```hcl
admin {
  listen       = "unix:///run/fileshare/admin.sock"   # made 0600
  state_file   = "/var/lib/fileshare/shares.json"
  source_roots = ["/srv/images", "/data"]
}
```

블록을 작성하지 않으면 아무것도 수신하지 않습니다.

## 하는 일 {#what-it-does}

서비스는
[`fileshare.admin.v1.AdminService`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto)입니다.

| | |
|---|---|
| `CreateShare`, `UpdateShare`, `DeleteShare` | 이미지, 디렉터리 또는 (v0.21.0부터) 볼륨으로 공유를 정의하고, `read_only` 또는 `protocols`를 변경합니다 (소스는 바꿀 수 없습니다. 삭제하고 다른 공유를 만드십시오) |
| `DisableShare`, `EnableShare` | 공유를 오프라인으로 전환하거나 다시 되돌립니다 |
| `Grant`, `Revoke` | 주체에게 읽기 또는 쓰기 권한을 부여하거나 회수합니다 |
| `ListShares`, `GetShare` | 모든 공유와 그 제공 대상, 사용 가능한 프로토콜, 나머지 프로토콜이 제외된 이유 |
| `ListUsers`, `ListGroups` | 권한 부여가 지정할 수 있는 대상. 각 사용자는 자격 증명으로 응답할 수 있는 프로토콜과 함께 표시됩니다 |
| `ReloadDirectory` | 사용자 정보를 지금 다시 읽습니다 — [디렉터리 다시 읽기]({{< relref "/administration/reload.md" >}}) 참고 |
| `GetServerInfo` | 이름, 버전, 시작 시각, 세대 및 리스너 |
| `ListParents`, `CreateVolume`, `ResizeVolume`, `SnapshotVolume`, `DeleteVolume`, `GetVolume`, `ListVolumes` | 권한 있는 프로비저너를 통해 생성된 스토리지와 그로부터 제공되는 공유 — [볼륨]({{< relref "/administration/volumes.md" >}}) 참고 (v0.21.0부터) |

권한 부여는 **사용자**, **그룹** (구성 파일에서는 `@group`),
**`oidc:groups:` 값** 또는 **`oidc:user:` 이름**을 지정합니다. 구성 파일이 쓰는
어휘와 같으며,
[ID 공급자가 지정하는 사용자]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})를 참고하십시오.

`grpc.health.v1`은 같은 리스너에서 응답합니다. 블록에 `reflection = true`를
쓰면 `grpcurl`을 위한 gRPC 서버 리플렉션이 켜집니다.

### 모든 변경은 그 변경으로 제공 상태가 어떻게 되었는지 알려 줍니다 {#every-change-says-what-serving-it-did}

변경은 `Applied` 메시지로 응답합니다.

```proto
message Applied {
  uint64 generation = 1;          // the generation now being served
  uint64 connections_closed = 2;  // how many the previous one had open, and closed
}
```

그래서 모든 RPC는 공유 자체를 반환하지 않고 각자의 응답 메시지를 가집니다.
`.proto`는 AIP-131의 "리소스 반환" 대신 [buf](https://buf.build)의 `STANDARD`
린트 규칙을 따르며, 이는 fileshare CI의 `buf lint`로 검사됩니다. `Applied`를
싣는 것이 바로 이 래퍼입니다.

## 수신하는 위치와 호출할 수 있는 대상 {#where-it-listens-and-who-may-call-it}

{{< callout type="error" >}}
**TCP에서는 상호 TLS가 아니면 허용되지 않습니다**

`tls_cert_file`, `tls_key_file`, `client_ca_file` 세 가지 모두가 필요하며,
**루프백도 예외가 아닙니다**. 모든 로컬 사용자가 루프백에 접근할 수 있고, 이
API는 누가 누구의 파일을 읽을지를 결정하기 때문입니다. 유닉스 소켓은
**0600**으로 만들어지며, 그 권한이 곧 접근 제어입니다.
{{< /callout >}}

```hcl
admin {
  listen         = "127.0.0.1:7443"
  tls_cert_file  = "/etc/fileshare/admin/server.pem"
  tls_key_file   = "/etc/fileshare/admin/server.key"
  client_ca_file = "/etc/fileshare/admin/clients.pem"
  state_file     = "/var/lib/fileshare/shares.json"
  source_roots   = ["/srv/images"]
}
```

리스너는
[grpc-transports/control](https://github.com/grpc-transports/control)입니다. 각
변경은 누가 했는지와 함께 기록됩니다. 클라이언트 인증서의 CN 또는 소켓 피어의
uid입니다.

## HTTPS를 통한 접근, OIDC 토큰용 {#over-https-for-oidc-tokens}

v0.28.0부터 `admin` 블록에 `web` 블록이 있으면 같은 API(같은 상태, 같은 감사)가
HTTPS로도 제공됩니다. **Connect, gRPC-Web, gRPC가 하나의 리스너**에서 제공되며,
[`tls` 블록]({{< relref "/security/tls.md" >}})의 인증서를 사용합니다. 웹 및
네이티브 UI가 사용하는 경로입니다.

```hcl
admin {
  listen     = "unix:///run/fileshare/admin.sock"
  state_file = "/var/lib/fileshare/shares.json"
  web {
    listen = "0.0.0.0:8443"
    issuer "https://login.example.org" {
      audience = "fileshare-a"            # what THIS server is called there
      groups   = ["fileshare-admins"]
    }
    issuer "https://login.partner.example.org" {
      audience = "fileshare-a.partner"
      subjects = ["5b0c…"]                # a person is (issuer, sub)
    }
  }
}
```

| 필드 | |
|---|---|
| `listen` | TCP 주소. `tls` 블록이 없으면 거부됩니다 — 베어러 토큰은 비밀번호와 같은 효력을 가집니다 |
| `issuer "<url>"` | ID 공급자마다 블록 하나, 곧 그 `iss`. 두 번 지정하면 거부됩니다 |
| `audience` | 토큰이 지정해야 하는 `aud`: 해당 공급자에서 **이 서버를 부르는 이름**. 필수 |
| `subjects`, `groups` | 호출할 수 있는 대상: `sub` 또는 그룹 클레임의 그룹. 둘 중 적어도 하나 |
| `jwks_url` | OIDC 디스커버리를 건너뜁니다. 이를 게시하지 않는 공급자용 |
| `groups_claim` | 그룹을 담는 클레임. 기본값은 `groups` |

**응답을 받는 대상.** 한 발급자의 검증기가 호출의 베어러 토큰(서명, `iss`,
이 서버를 지정하는 `aud`, 시각)을 수락하고, **또한** 그 발급자의 블록이 토큰의
주체 또는 그 그룹 중 하나를 지정할 때 호출에 응답합니다. 한 발급자에 나열된
주체는 다른 발급자에서는 아무도 아닙니다. 사용자는 (발급자, `sub`)입니다.
검사는 메시지의 한 바이트도 디코딩하기 전에 헤더만으로 수행됩니다.

**서버마다 하나의 audience.** 여러 서버를 수신자로 하는 토큰은 그중 어느 서버든
다른 서버에 재전송할 수 있으므로(RFC 8707, RFC 9068), 각 서버는 자신만의
audience를 가지며, 여러 서버를 다루는 UI는 서버마다 토큰 하나를 요청합니다.

**거부된 호출자가 알 수 있는 것.** "거부됨"이 전부입니다. 이유와 이루어진 모든
변경은 `oidc=<issuer> <sub>` 형식으로 감사 출력에 기록됩니다.

**CORS 없음.** 브라우저는 이 API를 직접 호출하지 않습니다. 토큰을 보관하는 UI
자체의 서버를 통해 접근합니다. 요청은 1 MiB로 제한되며 연결의 모든 단계에
타임아웃이 있습니다. 각 발급자는 키를 읽기 위해 **시작할 때** 접속됩니다.
접속할 수 없는 발급자가 있으면 그 발급자를 지목하며 시작이 중단됩니다.

### 브라우저에서: 콘솔 {#in-a-browser-the-console}

[go-fileshare/console](https://github.com/go-fileshare/console)은 이 API를
호출하는 관리 콘솔입니다. 공유, 상세 정보, 권한 부여, 생성, 활성화, 비활성화,
삭제, 부여와 회수, 그리고 [볼륨]({{< relref "/administration/volumes.md" >}})
(생성, 크기 조정, 스냅샷, 공유, 삭제 -- 데이터는 절대 파기하지 않음)을 모든
서버에 걸쳐 한꺼번에 다룹니다. 브라우저에서 실행되며,
[go-fileshare/portal](https://github.com/go-fileshare/portal) 뒤에 위치합니다.
포털은 OpenID Connect로 사용자를 로그인시키고 토큰을 보관합니다. 토큰은 서버마다
하나이며 해당 서버만을 수신자로 합니다. 브라우저는 토큰을 보관하지 않습니다.

```hcl
# portal.hcl
server "paris" {
  url      = "https://fs-paris.example.org:8443"   # this web listener
  issuer   = "https://login.example.org"
  resource = "https://fs-paris.example.org/"       # this server's audience above
}
ui "console" { dir = "/usr/share/fileshare-console" }
```

같은 콘솔은 포털 없이 **네이티브**로도 실행됩니다. 공개 클라이언트로서
시스템 브라우저와 루프백 리스너를 통해 로그인하고
([go-authn/rp](https://github.com/go-authn/rp), RFC 8252), 각 서버를 해당
서버만을 위한 토큰으로 호출합니다. 서버 목록은 자체 `console.hcl`에 있으며,
`url`과 `resource`는 여기와 같습니다.

**파일 탐색기** [go-fileshare/explorer](https://github.com/go-fileshare/explorer)는
같은 포털 뒤에서 실행됩니다. 사용자가 이용할 수 있는 모든 공유에서 탐색,
업로드, 다운로드, 이름 변경, 복사, 삭제를 할 수 있습니다. 포털(`/dav/<server>/...`)을
통해 해당 서버만을 위한 토큰으로 각 서버의 WebDAV에 접근하며, 포털은 돌아온
내용을 **비활성 상태**로 제공합니다(`Content-Security-Policy: sandbox`, 첨부
파일). 공유에 있는 페이지가 포털의 권한으로 실행되는 일은 없습니다. fileshare
**v0.29.0** 이상이 필요하며, 이 버전의
[WebDAV]({{< relref "/protocols/webdav.md#the-root-and-uploads-in-pieces" >}})가
루트에서 공유를 나열하고 조각 단위 업로드를 받습니다.

```hcl
# portal.hcl, 동일한 서버
server "paris" {
  # ... 위와 같고, 이 서버의 WebDAV:
  dav_url = "https://fs-paris.example.org"
}
ui "explorer" { dir = "/usr/share/fileshare-explorer" }
```

탐색기도 콘솔처럼 포털 없이 **네이티브**(explorer v0.2.0)로 실행됩니다.
공개 클라이언트로서 시스템 브라우저로 로그인하고, 각 서버의 WebDAV를 해당
서버만을 위한 토큰으로 호출합니다. 서버 목록은 `explorer.hcl`에 있으며, 각
서버에는 `dav_url`(https만 허용)과 `oidc` 블록의 `resource`가 있습니다. 두
네이티브 클라이언트 모두 토큰을 가진 채 리디렉션을 따라가지 않습니다. Go의 HTTP
클라이언트는 같은 호스트로의 리디렉션이라면 스킴이 달라도 `Authorization`을
유지하는데, 잘못 구성된 프록시가 평문 `http` 주소로 `307`을 응답하면 토큰이
평문으로 전송되기 때문입니다.

**실패한 업로드는 이어집니다** (explorer v0.3.0). 파트 파일의 이름은 파일의
이름, 크기, 수정 시각에서 만들어지므로, 같은 파일을 다시 업로드하면 파트가
끝난 지점부터 이어집니다. 단, 파트의 마지막 64 KiB를 다시 읽어 파일의 내용과
일치하는 것이 확인된 경우에 한합니다. 다른 파일은 결코 이어지지 않습니다.
네트워크나 서버의 실패는 파트를 남기고, 거부(403, 가득 찬 공유인 507)는 파트를
제거합니다.

각 서버의 `audience`는 포털에서 그 서버에 대한 `resource`입니다. API를 사용하기
위해 자체 콘솔을 작성할 필요는 없습니다. 이 서버용 토큰을 가진 Connect,
gRPC-Web 또는 gRPC 클라이언트라면 무엇이든 사용할 수 있습니다.

## 건드리는 것과 건드리지 않는 것 {#what-it-will-and-will-not-touch}

**자신이 만든 공유를 관리합니다.** 구성에 작성된 공유도 같은 필드와 함께
나열되지만, 그 정의를 바꾸려는 변경은 `FAILED_PRECONDITION`으로 거부됩니다.
두 곳에 정의된 공유는 아무도 답하고 싶지 않은 질문이며, 그 공유가 정의된
곳은 파일이기 때문입니다. API의 공유는 원자적으로 기록되는 `state_file`에
있으며 다음 시작 때 다시 제공됩니다. `state_file`이 없는 블록이 거부되는
이유가 이것입니다.

**소스는 `source_roots` 아래에 있어야 합니다.** 링크를 따라가고 `..`를 제거해
해석한 경로가 기준이며, 보관되는 것도 해석된 경로입니다. `source_roots`가
없으면 API는 **공유를 아예 만들 수 없습니다**. 프로세스는 `/dev`와 `/etc`를
읽을 수 있지만, 이를 호출자에게 넘길 의도는 아무에게도 없었기 때문입니다.
검사는 시작할 때마다 다시 수행되며, `state_file`의 공유가 더 이상 어떤
루트 아래에도 없으면 그 공유를 지목하며 시작이 중단됩니다.

**모든 공유에는 적어도 하나의 권한 부여가 있습니다.** 권한 부여가 없는 공유는
인증하는 모든 사람에게 열려 있습니다. 파일은 의도적으로 그렇게 지정할 수
있지만, API 호출이 누락으로 그렇게 지정해서는 안 됩니다. 그래서
`CreateShare`에는 권한 부여가 필요하고, 마지막 권한 부여를 회수하는 것은
거부됩니다. 대신 공유를 삭제하십시오.

**프로토콜이 담을 수 있는 이름.** 공유 이름은 최대 80자이고, 공백으로 시작하거나
끝나지 않으며, `.` 또는 `..`이 아니고, 출력할 수 없는 문자와 `:*?"<>|{}%` 중
어느 것도 포함하지 않습니다. 주체에도 출력할 수 없는 문자가 없어야 하며, 한
공유는 최대 1000개의 권한 부여를 받습니다. 그 정도라면 훨씬 이전에 그룹이 할
일입니다. 이 제약들은 모두 v0.14.0 이전에는 수락되어 `state_file`에 기록된 뒤
시작할 때마다 치명적 오류를 일으켰습니다.

**어떤 공유도 포함해서는 안 되는 것**은 구성, 상태 파일, 그리고 이들이 지정하는
비밀 정보입니다. 그 안에 쓸 수 있는 사람은 누가 무엇을 할 수 있는지를 다시 쓸
수 있기 때문입니다. v0.17.0부터는 `authorized_keys_file`, `users` 블록의
`dsn_file`과 `bind_password_file`, `ssf` 블록의 `ca_file`, 그리고 DSN이 지정하는
sqlite 데이터베이스가 여기에 포함됩니다. 누군가의 `authorized_keys`에 자신의
키를 추가할 수 있는 쓰기 권한자는 그 사람으로 로그인할 수 있기 때문입니다.
**다른 공유를 포함하는 것도 허용되지 않습니다**
([디렉터리]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}) 참고).

**변경은 구성처럼 검사되고, 열리고, 기록된 뒤에 제공됩니다.** 서버가 이행할 수
없는 변경 — 열리지 않는 이미지, 디렉터리에 없는 이름 — 은 거부되며, 제공 중이던
것과 기록된 것은 그대로 유지됩니다.

## 공유 비활성화 {#disabling-a-share}

**비활성화는 Samba의 `available = no`와 같습니다.** 공유는 정의된 채로 남고
모든 연결 시도가 실패합니다. 열려 있던 연결은 닫히고 이미지나 디렉터리는
**놓아 주므로**, 오프라인인 동안 파일을 교체할 수 있습니다. `EnableShare`는
다시 제공하며, 소스를 더 이상 열 수 없으면 거부됩니다. `CreateShare`로 처음부터
비활성화된 공유를 만들 수도 있습니다.

다른 모든 변경과 달리 **구성에 있는** 공유에도 적용됩니다. 공유를 오프라인으로
전환하는 것은 정의가 아니라 운영이기 때문입니다. 또한 재시작 후에도 유지되며,
상태 파일에 보관됩니다. [`fileshare check`]({{< relref "/configuration/check.md" >}})는
오프라인인 공유를 나열합니다.

## 변경은 새로운 세대입니다 {#a-change-is-a-new-generation}

{{< callout type="error" >}}
**변경은 프로토콜 서버를 재시작하고 그 연결을 닫습니다**

SMB는 트리 연결마다 한 번 접속 가능 여부를 검사하고, SFTP는 로그인마다 한 번
사용자의 트리를 만듭니다. 따라서 권한 회수 후에도 살아남은 세션은 방금 박탈된
접근 권한을 유지하게 됩니다. "이미 연결된 사람을 제외하고 회수됨"은 회수가
아닙니다.
{{< /callout >}}

그래서 변경은 실행 중인 서버에 *적용*되지 않습니다. 재시작할 때와 같은 방식으로,
새 목록으로 만든 새 서버 — **세대** — 로 교체되며, 포트는 놓지 않습니다.
라이브러리 쪽에서 달리 할 수 있는 방법이 없었습니다. go-filesystems/smb에는
`Share`는 있지만 `Unshare`가 없고, nfs에는 `Export`는 있지만 `Unexport`가
없습니다.

포트는 계속 바인딩되어 있으므로, 변경 중에 접속하는 클라이언트는 거부되지 않고
**대기합니다**. 공유가 바뀌지 않은 이미지는 드라이버를 유지하며 두 번 열리지
않습니다. 클라이언트는 다시 연결하며, 이것이 모든 변경에서 치르는 대가입니다.
그래서 API는 필드마다가 아니라 호출마다 하나의 변경을 적용합니다.

## 제한 {#limits}

- `--isolate`는 아직 `admin` 블록이나 `metrics` 블록과 함께 쓸 수 없습니다.
  변경을 적용할 단일 프로세스가 없기 때문입니다. 이 조합은 거부됩니다.
- `-tags nogrpc`로 빌드한 바이너리에는 관리 API가 없으며, `admin` 블록이 있는
  구성은 관리 API 없이 제공되지 않고 **거부됩니다**. 이 태그는 11.7 MB를
  줄입니다. [필요한 것만 빌드하기]({{< relref "/operations/build-tags.md#the-admin-api-and-nogrpc" >}})를
  참고하십시오.

## `.proto` 변경 {#changing-the-proto}

`proto/` 아래의 Go 코드는 생성되어 커밋되므로 `go install`에 `protoc`가
필요하지 않습니다. `.proto`(관리 API의 것 또는 프로비저너의
`proto/fileshare/provision/v1/provision.proto`)를 변경한 후에는 fileshare CI가
고정한 버전(protoc 34.1, protoc-gen-go v1.36.12, protoc-gen-go-grpc v1.6.2)으로
둘 다 다시 생성하십시오. CI는 이를 다시 생성하여 커밋된 코드와 다르거나 생성된
코드가 커밋되지 않은 채 남아 있으면 실패합니다.

```sh
protoc -I proto --go_out=. --go_opt=module=github.com/go-fileshare/fileshare \
  --go-grpc_out=. --go-grpc_opt=module=github.com/go-fileshare/fileshare \
  proto/fileshare/admin/v1/admin.proto proto/fileshare/provision/v1/provision.proto
```
