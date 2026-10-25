---
title: "WebDAV — Basic 또는 베어러 토큰"
linkTitle: "WebDAV: Basic 또는 베어러 토큰"
weight: 20
description: "HTTP Basic 또는 OIDC 베어러 토큰을 쓰는 WebDAV, 필수인 TLS, 그리고 토큰이 여기서 사용자가 되는 방식."
tags: [프로토콜, webdav, oidc]
---

```hcl
tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true
}
```

HTTP Basic 또는 베어러 토큰이며, 파일 또는 ACME에서 가져온
[TLS]({{< relref "/security/tls.md" >}}) 위에서 사용합니다.

{{< callout type="error" >}}
**v0.9.0부터 평문은 허용되지 않습니다**

HTTP Basic은 모든 요청에서 비밀번호 그 자체이며, 베어러 토큰도 그와 같은
효력을 가집니다. 누군가 인증하는 상태에서 다른 머신이 접근할 수 있는 주소의
**TLS 없는** WebDAV는 **시작할 때 거부됩니다**. 이 페이지가 예전에 보여 주던
예제인 `serve "webdav" { addr = "0.0.0.0:8080" }`는 더 이상 시작되지 않습니다.
프록시가 앞단에서 TLS를 종료하는 경우에는 그렇게 지정하십시오.

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true
}
```

루프백과 인증할 사용자가 없는 서버는 영향을 받지 않습니다.
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})를 참고하십시오.
{{< /callout >}}

{{< callout type="info" >}}
**403이 아니라 404**

사용자가 이용할 수 없는 공유는 **404**로 응답합니다. 존재가 확인되지 않습니다.
{{< /callout >}}

## 토큰, 그것을 운반할 수 있는 유일한 프로토콜 위에서 {#a-token-over-the-one-protocol-that-can-carry-one}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
}
```

브라우저에는 토큰은 있고 비밀번호는 없습니다. 그래서 WebDAV는 `Authorization:
Bearer`를 수락하며, 로컬 사용자도 있는 경우에는 보내는 챌린지가 **둘 다**
제시하여 클라이언트가 응답할 수 있는 쪽을 고릅니다. 토큰은
[go-authn/oidc](https://github.com/go-authn/oidc)가 서명, 발급자, audience,
만료를 검증합니다.

{{< callout type="error" >}}
**베어러 토큰: WebDAV만 가능**

SMB는 NTLMv2로, SFTP는 키나 인증서로, NFS는 아무것도 없이, S3는 SigV4 서명으로
인증합니다. 이들 중 어느 것도 `Authorization` 헤더를 실을 곳이 없습니다. 이는
프로토콜에 관한 사실이지 이 프로그램의 한계가 아닙니다.

공급자의 보증은 **SFTP**에서는 대신 인증서로 도착합니다.
[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})를
참고하십시오. 공급자를 지정하면서 WebDAV도 그런 SFTP도 제공하지 않는 구성은
**시작되지 않고 거부됩니다**.
{{< /callout >}}

{{< callout type="error" >}}
**토큰은 공급자가 누군가를 누구로 생각하는지 알려 줍니다. 이 서버에 그 사용자를 위한 공유가 있다는 뜻은 아닙니다.**

여기의 어떤 소스도 모르는 이름에 대한 유효한 토큰은 거부됩니다. *당신을 모릅니다*를
안전하게 해석하면 *허용됨*이 아닙니다. 로컬 계정도 가진 이름이라고 해서 그 계정은
아니며, `local_names = true`가 공급자의 이름이 이 서버의 이름이라고 말하지 않는 한
(v0.20.0부터 기본값은 꺼짐) 그렇지 않습니다. 이것이 없으면 `oidc:` 규칙이 해당
사용자를 지정하지 않는 한 그런 토큰도 거부됩니다.
{{< /callout >}}

공급자가 **곧** 디렉터리인 사이트는 그렇게 지정합니다.

```hcl
oidc {
  issuer    = "https://login.example.org"
  audience  = "fileshare"
  trust_all = true      # everybody that provider vouches for, not just these people
}
```

여기에는 **로그인 흐름이 없습니다**. 리디렉션도, 클라이언트 시크릿도, 쿠키도
없습니다. 이것은 리소스 서버입니다.

토큰이 주는 **이름**은 `preferred_username`이거나 `username_claim`이 지정하는
클레임입니다. `preferred_username`이 없으면, 공급자가 `email_verified: true`라고
말하는 경우에만 이메일이고, 그렇지 않으면 `sub`입니다. `username_claim =
"email"`이면 이메일이 검증되지 않은 토큰은 WebDAV에서도, SFTP를 통한 opkssh
로그인에서도 똑같이 거부됩니다. 검증되기 전의 이메일은 사용자가 입력한 것에
불과하며, 이를 담은 서명된 토큰은 아무도 확인하지 않은 주소로 바인딩될 것입니다
(OpenID Connect Core 5.1, go-authn/oidc v0.2.0, fileshare v0.16.3).

공급자의 사용자 중 누가 공유를 받는지(그룹, 지정된 사용자, 기관 전체)는 `oidc:`
규칙과 `domains`로 기술합니다.
[ID 공급자가 지정하는 사용자]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})를
참고하십시오.

## 토큰은 다른 조치가 없으면 사용자보다 오래 살아남습니다 {#a-token-outlives-the-person-unless-something-says-otherwise}

토큰은 여기서 단독으로 검증되며, 사용자에게 그 사이에 무슨 일이 있었든 만료될
때까지 유효합니다. 어떤 폐기 목록도 이를 지정할 수 없습니다.
[`ssf` 블록]({{< relref "/security/shared-signals.md" >}})은 공급자의 CAEP
`session-revoked` 이전에 발급된(`iat`) 모든 토큰을 무효로 만듭니다.

## 루트, 그리고 조각 단위 업로드 {#the-root-and-uploads-in-pieces}

**루트는 공유를 나열합니다** (v0.29.0부터). `/`의 브라우저는 열 수 있는 공유의
페이지를 받고, `/`의 `PROPFIND`는 이를 컬렉션(`207`)으로 받으므로, 파일 관리자가
공유 하나가 아니라 서버 전체를 마운트할 수 있습니다. 깊이 `infinity`는 거부되며
(`403`, `propfind-finite-depth`, RFC 4918이 허용하는 대로), 루트에 쓰는 모든 것도
거부됩니다(`405`). v0.28까지는 `/`에 대한 모든 메서드가 페이지를 받았습니다.

**큰 업로드는 조각으로 보낼 수 있습니다** (v0.29.0부터). `PUT` 하나는 최대 1 GiB까지
메모리에 통째로 유지되므로, 스트리밍할 수 없는 클라이언트(브라우저,
[파일 탐색기]({{< relref "/administration/_index.md" >}}))는 빈 `PUT`을 보낸 다음,
직전 조각이 끝난 지점에서 정확히 시작하는 `Content-Range`와 함께 각 조각을 보내고,
마지막으로 실제 이름으로 `MOVE`합니다. 끝을 넘어서 시작하는 범위는 여전히
`416`입니다. 아무도 보내지 않은 바이트가 남게 되기 때문입니다.
