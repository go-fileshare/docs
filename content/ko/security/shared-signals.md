---
title: "어떤 목록도 다루지 않는 것의 폐기: 공유 신호"
linkTitle: "공유 신호"
weight: 40
description: "공유 신호 전송자의 CAEP session-revoked 이벤트로 액세스 토큰, OpenPubkey 인증서 및 기타 연합 자격 증명을 폐기하는 방법."
tags: [보안, 폐기, 공유 신호]
---

인증서에는 폐기 목록인 [KRL]({{< relref "/security/revocation-lists.md" >}})과
[CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})이 있습니다.
fileshare가 자체적으로 검증하는 **액세스 토큰**과 사용자 자신의 키가 서명한
**OpenPubkey 인증서**에는 없습니다. 이들은 그 사이에 사용자에게 무슨 일이 있었든
만료될 때까지 유효합니다.

v0.13.0부터 [OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html)가
[CAEP](https://openid.net/specs/openid-caep-1_0-final.html) **`session-revoked`**
이벤트로 이를 해결합니다.

```hcl
ssf {
  transmitter        = "https://bridge.example.org"          # its issuer
  audience           = "https://files.example.org"           # what this server is to it
  client_id          = "fileshare"                           # OAuth client credentials,
  client_secret_file = "/etc/fileshare/ssf.secret"           # scope "ssf" (or token_file)
  state_file         = "/var/lib/fileshare/revocations.json"
  max_age            = "10m"                                 # default; retain = "192h"
}
```

go-authn/bridge는 사용자나 IdP가 비활성화될 때 이 이벤트를 보내며, fileshare는
이를 **폴링**(RFC 8936, SSF의 기본값)하고 사용자별로 **언제** 일어났는지를
보관합니다.

## 그 이전에 발급된 모든 것은 모든 프로토콜에서 거부됩니다 {#everything-issued-before-is-refused-on-every-protocol}

그 시점부터 그 **이전에 발급된** 모든 연합 자격 증명은 거부되며, **그것으로 연
세션은 서비스가 중단됩니다**. KRL과 같은 관문을 거칩니다. "발급"이 무엇을
뜻하는지는 자격 증명이 무엇에 실려 있었는지에 따라 다릅니다.

| 프로토콜 | 자격 증명 | 발급 시각 |
|---|---|---|
| WebDAV | 액세스 토큰 | 해당 `iat` |
| SFTP | 공급자의 SSH CA가 서명한 인증서 | 유효 기간 시작(`ValidAfter`) |
| SFTP | OpenPubkey(opkssh) 인증서 | ID 토큰의 `iat` |
| NFS | 클라이언트 인증서([identity = "certificate"]({{< relref "/security/nfs-certificates.md" >}})) | 해당 `NotBefore` |

공급자가 **그 이후에** 발급하는 것은 사용자의 것입니다. 다시 활성화된 사용자는
차단되지 않습니다.

{{< callout type="info" >}}
**같은 초는 이전으로 간주합니다**

CAEP의 `event_timestamp`와 토큰의 `iat`는 정수 초 단위이고, 폐기 **시점 또는 그
이전에** 발급된 자격 증명은 거부됩니다. 따라서 폐기와 같은 초에 발급된 것도
거부되며, 비활성화 직후 다시 활성화된 사용자는 다음 초부터 유효한 자격 증명을
받습니다. 의도적으로 보수적인 쪽입니다. 알 수 없는 발급 시각은 "아주 오래
전"이 아니며, 그 사용자에 대한 폐기가 하나라도 있으면 거부됩니다.
{{< /callout >}}

## 이벤트가 가리키는 대상 {#who-an-event-is-about}

주체는 RFC 9493의 **`account`**(`acct:user@domain`, 공유가 사용하는 이름),
**`iss_sub`**, **`email`**, 또는 이들의 **`aliases`**입니다.

**통째로 비활성화된 IdP**는 이벤트에 IdP의 스코프를 담은 CAEP *tenant* 주체로
도착합니다. 이름이 그중 하나의 `@` 뒤에 오는 모든 사람(전체를 비교하므로
`evil-univ-a.fr`은 `univ-a.fr`이 아닙니다)이며 그 자격 증명이 그 이전에 발급된
경우 거부되고, **공급자가 더 이상 기억하지 못하는 사용자도 포함됩니다**.

## 수신자가 인증하는 방법 {#how-the-receiver-authenticates}

**OAuth 클라이언트 자격 증명**(RFC 6749 §4.4, 스코프 `ssf`)으로 합니다.
`client_id`와 `client_secret_file`을 사용하고, 토큰은 전송자 자체의 토큰
엔드포인트(OpenID 구성에서 찾거나 `token_url`)에서 가져와 만료 전에 갱신합니다.
`token_file`은 장기 베어러 토큰을 발급하는 전송자를 위해 대신 쓸 수 있습니다.
둘 중 하나여야 하며, 둘 다이거나 둘 다 없으면 거부됩니다.

| 필드 | 기본값 | |
|---|---|---|
| `transmitter` | | 전송자의 발급자, **HTTPS만 허용** |
| `audience` | | 이 서버가 전송자에게 어떤 존재인지. 필수이며, 없으면 다른 수신자에게 보낸 SET가 여기서 신뢰될 수 있습니다 |
| `client_id`, `client_secret_file` | | OAuth 클라이언트 자격 증명, 함께 지정 |
| `token_url` | 디스커버리 | HTTPS만 허용. 아니면 비밀이 평문으로 네트워크를 건넙니다 |
| `token_file` | | 클라이언트 자격 증명 대신 쓰는 장기 토큰 |
| `state_file` | | **필수**: 폐기가 기록되는 위치 |
| `ca_file` | 시스템의 것 | 전송자의 기관을 고정 |
| `max_age` | `10m` | 전송자로부터 소식이 없는 시간이 이를 넘으면 연합 자격 증명이 거부됨. 최소 1분 |
| `retain` | `192h` | 폐기를 보관하는 기간. 최소 169h(v0.14.0부터) |

폐기는 **확인 응답을 보내기 전에 기록되므로**, 확인된 폐기는 재시작 후에도
유지됩니다. 또한 `retain` 동안 보관되며, 이는 그것이 무효화할 수 있는 어떤
자격 증명보다 깁니다(192h는 공급자가 발급하는 가장 수명이 긴 자격 증명인 168h SSH
인증서보다 깁니다).

폐기할 연합 대상이 없는 `ssf` 블록, 곧 `oidc` 블록도 NFS
`identity = "certificate"`도 없는 경우는 거부됩니다.

전송 계층은 [github.com/hstern/go-ssf](https://github.com/hstern/go-ssf)로,
디스커버리, RFC 8936 폴러, SET의 JWS 계층입니다. 이벤트가 **의미하는** 바는
fileshare의 몫입니다.

## fail closed입니다 {#it-fails-closed}

{{< callout type="error" >}}
**전송자가 침묵하는 동안 연합 자격 증명은 거부됩니다**

KRL과 마찬가지로, 전송자가 `max_age` 안에 폴링에 응답하지 않으면 **모든 연합 자격
증명**이 거부됩니다. "폐기가 도착하지 않음"과 "도착할 수 없었음"은 똑같아 보이기
때문입니다. 알림을 설정할 메트릭은 `fileshare_ssf_last_heard_seconds`이고,
`fileshare_ssf_revoked_subjects`는 보관 중인 폐기의 수입니다.
[상태 점검과 메트릭]({{< relref "/administration/health.md#what-to-alert-on" >}})을
참고하십시오.
{{< /callout >}}

## 처음부터 끝까지 검증 {#checked-end-to-end}

go-authn/bridge의 상호 운용 레인
([#12](https://github.com/go-authn/bridge/pull/12),
[#17](https://github.com/go-authn/bridge/pull/17))이 fileshare v0.13.0을 실제
bridge에 대해 판정합니다. KRL, NFS의 CRL, SSF로 폐기된 WebDAV 토큰, 스코프로
비활성화된 IdP, 그리고 SSF를 통한 opkssh로, 어떤 폐기 목록도 닿지 않는
경우입니다.
