---
title: "TLS와 ACME 인증서"
weight: 10
description: "파일 또는 ACME CA의 인증서로 WebDAV, S3, NFS를 TLS 위에서 제공하는 방법, 그리고 비밀번호를 쓰는 WebDAV의 평문이 거부되는 이유."
tags: [보안, tls, acme]
---

v0.9.0부터 **WebDAV와 S3는 HTTPS로, NFS는 RPC-with-TLS**(RFC 9289)로 제공됩니다.
`serve` 블록의 `tls = true`와 인증서의 출처를 지정하는 하나의 `tls` 블록을
사용합니다.

## 파일에서 {#from-files}

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

파일은 **변경되면 다시 로드되므로**, 다른 도구가 기록한 갱신본이 재시작 없이
반영됩니다. 인증서는 [go-authn/servercert](https://github.com/go-authn/servercert)의
것입니다.

## ACME CA에서 {#from-an-acme-ca}

기본값은 Let's Encrypt이며, 또는 HARICA를 통한 GÉANT TCS처럼 **외부 계정
바인딩**으로 여러분을 인식하는 CA를 쓸 수 있습니다.

```hcl
tls {
  acme {
    directory_url     = "https://acme-v02.harica.gr/acme/<uuid>/directory"   # the Server URL cm.harica.gr shows
    domains           = ["files.example.org"]
    cache_dir         = "/var/lib/fileshare/acme"
    eab_key_id        = "…"
    eab_hmac_key_file = "/etc/fileshare/tls/eab.key"   # a secret: a file, never the config
  }
}
```

| 필드 | |
|---|---|
| `directory_url` | CA의 ACME 디렉터리. 비어 있으면 Let's Encrypt |
| `domains` | 요청할 이름(필수) |
| `cache_dir` | 계정과 인증서를 보관하는 위치(필수) |
| `email` | 계정의 연락처, 선택 사항 |
| `eab_key_id`, `eab_hmac_key_file` | 외부 계정 바인딩. HMAC 키는 비밀이므로 파일에서 읽으며, CA가 발급하는 그대로 base64url입니다(표준 base64도 읽습니다). 그 밖의 형식은 거부됩니다 |
| `http_challenge` | http-01에 응답할 위치. 예: `"0.0.0.0:80"` |

### ACME가 동작할 수 있는지는 CA가 이름을 확인하는 방식으로 결정됩니다 {#whether-acme-can-work-is-decided-by-how-the-ca-checks-the-name}

| 챌린지 | CA가 접속하는 곳 | 따라서 |
|---|---|---|
| tls-alpn-01 (RFC 8737) | **443** 포트 | TLS 프로토콜이 443에서 제공되어야 합니다 |
| http-01 (RFC 8555 §8.3) | **80** 포트 | `http_challenge = "0.0.0.0:80"`이 응답합니다. 또는 80 포트가 포워딩되는 주소 |
| 없음 | 없음 | 도메인이 **사전 검증된** CA 계정은 챌린지를 요구하지 않습니다. GÉANT TCS(OV 인증서)의 HARICA Enterprise **Admin** 계정이 그렇습니다. 그래서 **외부에서 접근할 수 없는** 서버도 인증서를 받습니다 |

마지막 행이 캠퍼스 네트워크 안의 파일 서버에 중요한 경우입니다. 기관이 HARICA에서
이미 도메인을 검증한 GÉANT TCS 계정이 있으면 인터넷에 포트를 전혀 열 필요가
없습니다.

#### HARICA (GÉANT TCS): 어느 계정인가, 그리고 CAA {#harica-gant-tcs-which-account-and-caa}

HARICA의 ACME 계정은 두 종류이며, 그중 하나만 챌린지를 건너뜁니다.

- **Enterprise Admin** 계정은 기관이 검증한 도메인에 대해 **OV** 인증서를 발급합니다.
  **챌린지가 없으며**, 외부에서 접근할 수 없는 서버를 위한 계정입니다.
- **Enterprise User** 계정은 **DV** 인증서를 발급하며, DV는 CA가 주문할 때마다
  위의 tls-alpn-01 또는 http-01로 이름을 확인한다는 뜻입니다.

어느 쪽이든 도메인에 **CAA** 레코드가 있다면 `harica.gr`을 허용해야 하며, 그렇지
않으면 주문이 거부됩니다.

#### 인증서를 요청하는 시점 {#when-the-certificate-is-asked-for}

**시작할 때**(v0.27.0부터): 리스너가 올라오면 구성된 모든 도메인의 인증서를
가져오거나 캐시 디렉터리에서 다시 읽습니다. 실패해도 서버는 멈추지 않고 계속
제공하며 1분 후에 다시 시도하고, 최대 1시간까지 두 배씩 늘립니다. 호스트 이름을
지정하는 첫 TLS 연결도 어쨌든 다시 요청합니다. 인증서는 만료 30일 전에, 또는 그것이
더 짧다면 수명의 마지막 3분의 1 시점에 갱신됩니다.

**SNI를 보내지 않는 클라이언트**, 곧 IP 주소로 접속하는 클라이언트에는
**첫 번째** 도메인의 인증서가 제공됩니다(v0.27.0부터). 이전에는 핸드셰이크가
실패했습니다. 클라이언트는 여전히 그 인증서를 자신이 접속한 주소와 대조해 검사하므로,
그 이름을 신뢰하라고 안내받은 클라이언트에게는 도움이 되고 아무도 속이지 않습니다.

{{< callout type="info" >}}
**ALPN**

ACME를 쓰면 서버의 ALPN 목록에 `acme-tls/1`이 있고, Go 서버는 자신의 목록과
공통점이 없는 클라이언트를 거부합니다. 그래서 WebDAV와 S3에는 `http/1.1`을
**덧붙이고**(모든 브라우저와 WebDAV 클라이언트가 이를 제시합니다), NFS에는
RFC 9289 §5.2가 RPC-with-TLS를 식별하는 `sunrpc`을 덧붙입니다. `h2`는 아닙니다.
이 리스너에서는 HTTP/2를 제공하지 않으며 제공한다고 하면 거짓이 됩니다.
`acme-tls/1`은 모든 리스너에 남아 있으므로 CA가 도달하는 어디서든 tls-alpn-01에
응답합니다.
{{< /callout >}}

## WebDAV는 평문으로 제공되지 않습니다 {#webdav-is-not-served-in-the-clear}

{{< callout type="error" >}}
**v0.9.0의 호환성 파괴: 접근 가능한 주소에서 비밀번호를 쓰는 WebDAV는 거부됩니다**

HTTP Basic은 모든 요청에서 base64로 인코딩된 비밀번호 그 자체이며, 베어러 토큰도
그와 같은 효력을 가집니다. 따라서 누군가 인증하는 상태에서 다른 머신이 접근할 수
있는 주소에서 **TLS 없이** WebDAV를 제공하는 구성은 경고가 아니라 **거부됩니다**.
경고는 스크롤되어 사라지지만 유출된 비밀번호는 돌아오지 않기 때문입니다.

```
webdav on 0.0.0.0:8080 would carry passwords in the clear: serve it with
tls = true, or -- when TLS is terminated in front of it, by a proxy -- say
plaintext = true
```
{{< /callout >}}

v0.9.0 이전에 `serve "webdav" { addr = "0.0.0.0:8080" }`로 시작되던 구성은 그
이후에는 시작되지 않습니다. 해결 방법은 두 가지입니다.

```hcl
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true            # with a tls block
}
```

또는 리버스 프록시가 앞단에서 TLS를 종료하고 평문으로 전달하는 경우입니다.

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true       # TLS is terminated in front of this, on purpose
}
```

**루프백** 주소(`127.0.0.1`, `::1`, `localhost`)와 **인증할 사용자가 없는**
서버(`user`, `users`, `oidc` 블록이 모두 없음)는 영향을 받지 않습니다. `addr`이 없는
`serve` 블록은 루프백에 위치하므로 역시 영향을 받지 않습니다.

`plaintext`는 비밀번호를 보내는 프로토콜인 WebDAV 전용이며, `plaintext`와 `tls`를
함께 쓰면 거부됩니다. 둘 중 하나여야 합니다.

## `check`가 알려 주는 것 {#what-check-says}

[`fileshare check`]({{< relref "/configuration/check.md" >}})는 프로토콜별로 무엇이
암호화되고 무엇이 **의도적으로** 평문인지 출력합니다.

```
webdav: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem
nfs: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem; clients present one signed by /etc/fileshare/tls/clients.pem (the machine, not the person)
```

그리고 프록시 뒤의 WebDAV에 대해서는 다음과 같이 출력합니다.

```
webdav: in the clear on 10.0.0.5:8080, on purpose (plaintext = true): TLS must be terminated in front of it
```

## 프로토콜별 {#per-protocol}

| 프로토콜 | `tls = true` | |
|---|---|---|
| WebDAV | 가능 | HTTPS |
| S3 | 가능 | HTTPS |
| NFS | 가능 | RFC 9289, 아래 참고 |
| SMB | **거부** | SMB 3은 자체 키로 암호화합니다 |
| SFTP | **거부** | SFTP는 SSH입니다 |

## TLS 위의 NFS는 사용자가 아니라 머신을 증명합니다 {#nfs-over-tls-proves-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

`client_ca_file`은 클라이언트가 그 기관의 인증서, 곧 **어느 호스트가 마운트하는지**를
제시하게 하며, RFC 9289는 사용자 인증을 이전 그대로 둡니다. 그 안의 uid는 여전히
`AUTH_SYS`가 주장하는 값입니다. 따라서 사용 가능한 사용자를 지정한 공유는
`kerberos` 블록이 없으면, 또는 같은 인증서의 다른 용도인
[인증서 기반 ID]({{< relref "/security/nfs-certificates.md" >}})가 없으면 NFS에서
**여전히 거부됩니다**.

TLS는 **제공될 뿐 필수가 아닙니다**. 이를 요청하지 않는 클라이언트에게도 열린
공유는 계속 제공됩니다. `tls = true` 없는 `client_ca_file`은 아무것도 검사하지
않으므로 거부되며, NFS 외의 프로토콜에 쓴 `client_ca_file`도 거부됩니다.

## 그 밖에 거부되는 것 {#what-else-is-refused}

- `tls` 블록이 없는 `tls = true`: 인증서의 출처를 알려 주는 것이 없습니다.
- 어떤 `serve` 블록도 사용하지 않는 `tls` 블록: 아무것도 사용하지 않게 됩니다.
- 수신할 주소가 아닌 `http_challenge`.
