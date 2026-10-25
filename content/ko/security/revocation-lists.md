---
title: "SSH 인증서 폐기: KRL"
weight: 20
description: "로그인 시와 모든 SFTP 작업마다 검사되는, 서명되고 날짜가 기록된 OpenSSH KRL로 ID 공급자의 SSH 인증서를 폐기하는 방법."
tags: [보안, 폐기, sftp, ssh]
---

ID 공급자의 SSH CA가 서명한 인증서
([공급자가 보증하는 사용자를 위한 SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}) 참고)는
로그인할 때 검사됩니다. go-authn/bridge에서 누군가를 비활성화하면 그 사용자의
토큰이 폐기되고 애플리케이션 비밀번호가 삭제되며, 이는
[디렉터리 다시 불러오기]({{< relref "/administration/reload.md" >}})로 여기에 반영됩니다.
그러나 목록이 없으면 이미 발급된 인증서는 만료될 때까지 SFTP를 열 수 있고, 이미
열린 SFTP 세션은 열린 채로 남습니다.

v0.12.0부터 목록이 이를 막습니다.

```hcl
oidc {
  issuer          = "https://login.example.org"
  audience        = "fileshare"
  ssh_ca_file     = "/etc/fileshare/bridge-ca.pub"
  ssh_krl_url        = "https://bridge.example.org/ssh/krl"   # or ssh_krl_file
  ssh_krl_state_file = "/var/lib/fileshare/ssh.krl.state"    # the order, across a restart
  ssh_krl_max_age    = "1h"                                  # default; ssh_krl_refresh = "1m"
}
```

go-authn/bridge는 폐기한 SSH 인증서(비활성화된 사용자, 비활성화된 IdP)를 OpenSSH
**KRL**(`PROTOCOL.krl`, `sshd`의 `RevokedKeys`가 읽는 목록)로 게시하며, fileshare는
그 사본을 보관하고 [go-authn/krl](https://github.com/go-authn/krl)로 읽습니다.

## 로그인 시와 세션 도중의 거부 {#refused-at-login-and-in-the-middle-of-a-session}

폐기된 인증서는 로그인할 때 거부되며, **그 인증서로 이미 연 세션은 더 이상
서비스되지 않습니다**. SSH는 인증서를 핸드셰이크 때 한 번만 검사하므로, 연합 SFTP
세션의 모든 작업은 **열린 파일을 포함해** KRL을 다시 조회합니다. 이 서버의 목록
사본이 바뀐 뒤의 첫 작업은 거부됩니다.

## 필드 {#fields}

| 필드 | 기본값 | |
|---|---|---|
| `ssh_krl_url` | | 목록을 가져오는 위치, **HTTPS만 허용** |
| `ssh_krl_file` | | 또는 절대 경로의 파일 — 둘 중 하나이며 둘 다는 안 됩니다 |
| `ssh_krl_ca_file` | 시스템의 것 | 목록의 HTTPS 서버를 확인할 기관을 고정 |
| `ssh_krl_refresh` | `1m` | 가져오는 주기, 최소 1초 |
| `ssh_krl_max_age` | `1h` | 마지막 정상 사본이 얼마나 오래되면 그 목록이 관할하는 인증서가 거부되는지 |
| `ssh_krl_state_file` | | 검증된 마지막 목록을 보관하는 위치. 재시작 후에도 순서를 기억하게 합니다 |

`refresh`보다 짧은 `max_age`는 거부됩니다. 모든 사본이 다음 가져오기 전에 낡게
되기 때문입니다. `ssh_ca_file`이 없는 KRL도 거부됩니다. KRL은 공급자의 인증서를
폐기하는 것인데, 그 CA가 없으면 폐기할 인증서가 없기 때문입니다.

## fail closed입니다 {#it-fails-closed}

{{< callout type="error" >}}
**KRL을 가져올 수 없는 동안 공급자의 인증서는 거부됩니다**

목록을 가져올 수 없거나 마지막 정상 사본이 `ssh_krl_max_age`보다 오래되면,
공급자의 CA가 서명한 **모든** 인증서가 거부됩니다. 목록에 접근할 수 없을 때 모든
것을 통과시키는 폐기 검사는 목록을 막을 수 있는 공격자에게 무력화되는 검사이기
때문입니다. 그러면 목록은 그것이 관할하는 로그인만큼 안정적으로 제공되어야
합니다. `max_age`에 도달하기 전에 알림을 설정할 메트릭은
`fileshare_revocation_list_age_seconds{list="ssh_krl"}`입니다.
[상태 점검과 메트릭]({{< relref "/administration/health.md#what-to-alert-on" >}})을
참고하십시오.
{{< /callout >}}

## 서명되고 날짜가 기록됨, `ssh_krl_url`에서 {#signed-and-dated-from-ssh_krl_url}

v0.15.0부터 **`ssh_krl_url`의 KRL은 서명되어 있고 만료 시점을 명시해야 합니다**.
[go-authn/revocation](https://github.com/go-authn/revocation)이 설명하고
go-authn/bridge가 v0.10.0부터 제공하는 방식입니다.

- **서명:** `<url>.sig`는 공급자의 SSH CA(`ssh_ca_file`이며, 따라서
  `ssh_krl_url`에는 필수)가 `krl@go-authn.github.io` 네임스페이스로 만든 분리형
  SSHSIG 서명입니다. 목록의 ETag에 대한 `If-Match`와 함께 가져오며, 그사이에
  목록이 재발급되었으면 즉시 다시 가져옵니다.
- **만료:** `expires@go-authn.github.io` 확장. 이를 넘기면 `304`가 아무리 최근에
  확인해 주었더라도 목록은 최신이 아닙니다.

HTTPS(`ssh_krl_ca_file`이 기관을 고정)는 응답한 서버를 인증하며 목록을 인증하지
않습니다. 미러, 캐시 또는 침해된 웹 서버가 빈 목록을 제공할 수 있고, KRL 자체에는
낡았음을 알려 주는 것이 없습니다. 검증되지 않는 목록은 거부되고 마지막 정상 사본이
유지됩니다. `ssh_krl_file`로 직접 놓은 목록은 그것을 보관하는 파일 시스템만큼
신뢰되며, 서명도 만료도 필요하지 않습니다. 목록에 있다면 그대로 존중됩니다.

{{< callout type="warning" >}}
**v0.15.0으로 업그레이드할 때**

서명되지 않은 KRL이나 만료가 없는 KRL을 제공하는 `ssh_krl_url`은 더 이상
수락되지 않으며, 얻을 수 없는 다른 모든 목록과 마찬가지로 공급자의 인증서가
거부됩니다. 먼저 go-authn/bridge v0.10.0 이상을 실행하십시오.
{{< /callout >}}

## 재시작해도 결코 거꾸로 가지 않음 {#never-backwards-across-a-restart-too}

v0.16.0부터 보유한 목록보다 오래된 목록은 거부됩니다. 더 낮은 버전(KRL의 버전,
CRL의 번호), 또는 같은 버전이지만 더 일찍 발급된 목록입니다. `ssh_krl_state_file`
([NFS]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})에서는
`crl_state_file`)이 있으면 검증된 마지막 목록이 디스크에 보관되고, 시작할 때
다시 검증되며, 다음 목록의 순서를 정합니다. 이것만으로는 목록을 가져오기 전까지
최신으로 간주되지 않습니다. 이것이 없으면 재시작할 때 순서를 잊어버리며,
fileshare는 시작할 때 이를 알립니다. 여전히 서명되어 있고 만료되지 않은 더 오래된
목록이 수락될 수 있기 때문입니다.

이 파일들은 `ssh_ca_file`, CA 파일, 목록 파일, 그리고 (v0.17.0부터)
`authorized_keys_file`, `dsn_file`, `bind_password_file`, `ssf`의 `ca_file`과
마찬가지로 **공유 안에 있어서는 안 됩니다**. 그곳에 쓸 수 있는 사람이 누가 들어올
수 있는지를 결정하게 되기 때문입니다.

## fileshare 옆의 일반 `sshd` {#a-plain-sshd-next-to-fileshare}

같은 검증은 fileshare 없이도 사용할 수 있습니다. go-authn/revocation의
`authn-revokd`는 목록을 가져와 검증되는 것만 보관하고, 그 순서를 디스크에
유지하며, `sshd`를 위한 `RevokedKeys`를 작성합니다. 목록이 만료되면 CA 자체를
폐기하는 목록을 작성하므로 `sshd`도 fail closed가 됩니다.

## 다루지 않는 것 {#what-it-does-not-cover}

**OpenPubkey(opkssh) 인증서는 어떤 KRL에도 없습니다**. 사용자 자신의 키 외에는
그것을 발급한 주체가 없기 때문입니다. 이 인증서는 `opkssh_max_age`로 한도가
정해지고 [공유 신호]({{< relref "/security/shared-signals.md" >}})로 회수됩니다.
WebDAV의 액세스 토큰도 마찬가지입니다.

같은 메커니즘, 곧 가져오고 보관하며 fail closed인 방식이
[인증서 기반 NFS ID]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})의
**X.509 CRL**을 실어 나릅니다.
