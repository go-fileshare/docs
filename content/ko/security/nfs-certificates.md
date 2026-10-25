---
title: "인증서 기반 ID를 쓰는 NFS"
weight: 30
description: "X.509 클라이언트 인증서로 지정되고 CRL로 폐기되는 사용자에게 NFS로 제한된 공유를 제공하는 방법, 그리고 공유 클라이언트에서 이것이 보장할 수 없는 것."
tags: [보안, nfs, tls, 폐기]
---

NFSv3는 단독으로는 누구도 인증하지 않으며, 그래서 사용 가능한 사용자를 지정한
공유는 사용자를 구별할 수 있는 무언가가 없는 한
[NFS로 내보내지 않습니다]({{< relref "/protocols/nfs.md" >}}). `kerberos` 블록이 그
해법 중 하나입니다. v0.12.0부터는 **사용자를 지정하는** 클라이언트 인증서가 또
다른 해법입니다.

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file; required
  crl_state_file = "/var/lib/fileshare/nfs.crl.state"     # the order, across a restart
}
```

## 인증서가 담는 것 {#what-the-certificate-carries}

ID 공급자 로그인 후 go-authn/bridge는 FreeBSD의 `rpc.tlsservd -u`가 읽는 방식으로
사용자를 지정하는 **단기 X.509 클라이언트 인증서**를 발급합니다.

- 사용자: SubjectAltName **otherName `1.3.6.1.4.1.2238.1.1.1`**, 곧 `user@domain`
  형식의 UTF8String
  ([draft-cel-nfsv4-rpc-tls-othername](https://www.ietf.org/archive/id/draft-cel-nfsv4-rpc-tls-othername-04.html)이
  설명하며, 이 OID는 아직 할당되지 않았습니다)
- 그룹: 그룹마다 SubjectAltName URI 하나,
  `tag:go-authn.github.io,2026:group:<group>` — 바로 이 용도로 만들어진
  **tag URI**(RFC 4151: 등록 불필요)입니다. FreeBSD의 `rpc.tlsservd`는 첫 번째 ID
  otherName을 읽고 다른 모든 SAN 항목은 건너뜁니다.

{{< callout type="info" >}}
**자체 확장이 아니라 tag URI를 쓰는 이유**

UUID에서 파생한 아크(`2.25.<128비트 숫자>`) 아래에 자체 확장을 설계하고
**측정했습니다**. Go의 `x509.ParseCertificate`는 인증서 전체를 거부합니다
(*malformed extension OID field*). 그곳에서 ASN.1 객체 식별자 아크는 `int`이기
때문입니다. 모든 Go TLS 서버가 이 인증서를 돌려보냈을 것입니다. URI SAN은 OID가
필요 없으며 모든 파서가 읽습니다. `urn:x-…`도 아닙니다. RFC 8141이 실험용 URN
네임스페이스를 폐지했고, 엄격한 파서는 이를 거부하기 때문입니다.
{{< /callout >}}

## 모든 호출에서 확인하는 것 {#what-every-call-is-asked}

`identity = "certificate"`이면, 사용자를 지정하는 공유는 **kerberos 없이** NFS로
제공됩니다. 그 공유에 대한 모든 호출은 TLS로 도착해야 하며, 인증서는 다음을
만족해야 합니다.

1. **CRL에 없음**
2. **정확히 한 명**의 사용자를 지정함
3. 이 서버가 **토큰을 허가하는 방식으로** 허가하는 사용자를 지정함 — 규칙,
   `trust_all`, `domains`.
   [ID 공급자가 지정하는 사용자]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})를
   참고하십시오.
4. **공유가 허용하는** 사용자를 지정함

그리고 [`ssf` 블록]({{< relref "/security/shared-signals.md" >}})이 있으면, 인증서는
해당 사용자에 대한 모든 폐기 **이후에** 발급(`NotBefore`)되어야 합니다.

## 폐기: CRL {#revocation-the-crl}

**CRL은 필수입니다** — `crl_url` 또는 `crl_file`. 아무것도 폐기할 수 없는 사용자의
인증서는 사용자가 제거된 뒤에도 그 수명 전체만큼 살아남습니다.

| 필드 | 기본값 | |
|---|---|---|
| `crl_url` | | HTTPS만 허용 |
| `crl_file` | | 또는 절대 경로 — 둘 중 하나 |
| `crl_ca_file` | 시스템의 것 | CRL의 HTTPS 서버를 확인할 기관을 고정 |
| `crl_refresh` | `1m` | 가져오는 주기 |
| `crl_max_age` | `1h` | 마지막 정상 사본이 얼마나 오래되어도 되는지 |
| `crl_state_file` | | 검증된 마지막 CRL을 보관하는 위치. 재시작 후에도 어느 쪽이 더 새로운지 기억하게 합니다 |

[KRL]({{< relref "/security/revocation-lists.md" >}})과 같은 방식으로 가져와 보관하며
같은 방식으로 **fail closed**입니다. 자체 **`NextUpdate`**가 지난 CRL은
`crl_max_age`가 허용하는 값과 관계없이 알 수 없는 것으로 간주합니다. CRL은 클라이언트
CA가 서명해야 합니다. 누구나 쓸 수 있는 CRL은 원하는 것을 마음대로 폐기할 수 있고,
더 나쁘게는 폐기를 취소할 수도 있습니다. CRL에 접근할 수 있으면 **변경 후 다음
호출이 거부됩니다**. 인증서의 수명이 폐기의 한계가 되는 것은 CRL에 접근할 수 없을
때뿐입니다.

v0.16.2부터 서명은 CRL을 파싱하기 **전에** 원시 바이트에 대해
[go-authn/revocation](https://github.com/go-authn/revocation)을 통해 검사합니다.
먼저 파싱하면 아무나 서명한 큰 CRL이 거부되기 전에 메모리와 시간을 소모했습니다.
이 서버가 완전한 목록으로 읽을 수 없는 것도 거부됩니다. 델타 CRL, 알 수 없는 크리티컬
확장 또는 항목 확장, CRL 번호 없음, `NextUpdate` 없음입니다. 보유한 것보다 번호가
낮은 CRL, 또는 번호가 같지만 더 일찍 발급된 CRL은 오래된
[KRL]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}})과
마찬가지로 거부됩니다.

`identity = "certificate"` 없이 CRL 필드를 쓰면 거부되며,
`tls = true`와 `client_ca_file` 없이 `identity = "certificate"`를 쓰는 경우도
마찬가지입니다. ID는 클라이언트가 제시하는 인증서 안에 있기 때문입니다.

## 보장할 수 없는 것 {#what-it-cannot-promise}

{{< callout type="error" >}}
**Linux 클라이언트의 인증서는 사용자가 아니라 마운트에 속합니다**

서버는 **연결의** 인증서를 보며, Linux 클라이언트는 이를 마운트에 연결합니다
(`tlshd`, `mount -o xprtsec=mtls,...`의 키링 일련번호). **해당 마운트의 모든
사용자는 인증서가 지정하는 사용자로 동작합니다.** 한 사람이 사용하는
워크스테이션에서는 정확히 그 사람이지만, 여러 사람이 로그인하는 머신에서는
마운트한 사람입니다. 그런 곳에서는 `sec=krb5`를 사용하십시오.

RFC 9289만으로는 사용자 인증을 보장한다고 하지 않는 이유가 이것이며, 단순한
[TLS 위의 NFS]({{< relref "/security/tls.md#nfs-over-tls-proves-the-machine-not-the-person" >}})가
여전히 머신만 증명하는 이유이기도 합니다.
{{< /callout >}}

## 실제 Linux 클라이언트로 측정 {#measured-with-a-real-linux-client}

go-filesystems/nfs의 CI에서 커널 6.17, ktls-utils 0.9로 측정했습니다.

- **MOUNT의 `MNT`는 `xprtsec=`이 무엇이든 평문으로 도착합니다**. 커널의 마운트
  클라이언트에는 TLS가 없습니다. 응답은 하되, TLS 없이 이루어진 모든 NFS 호출은
  이후 거부되므로, 공유가 허용하지 않는 사용자는 `mount`가 아니라 첫 접근에서
  *access denied*를 봅니다. **공유의 어떤 것도 평문으로 건너지 않습니다.**
- `tlshd`는 이 서버의 인증서를 **시스템** 신뢰 저장소로만 검사하고(`x509.truststore`는
  무시함), 클라이언트의 인증서와 키가 **root 소유이고 키의 모드가 600**이기를
  요구합니다. 그렇지 않으면 어느 쪽도 지목하지 않고 *gnutls: Error in the
  certificate (-43)*으로 실패합니다.
