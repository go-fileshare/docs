---
title: "프로토콜이 보장할 수 있는 것"
linkTitle: "프로토콜"
weight: 20
description: "각 프로토콜이 요청한 사용자를 구별할 수 있는지 여부와, 그것이 어떤 공유를 제공할 수 있는지에 미치는 영향."
tags: [프로토콜]
---

프로토콜들은 접근 제어에 필요한 한 가지, 곧 서버가 요청하는 사용자가 **누구인지**
구별할 수 있는가에 대해 의견이 일치하지 않습니다.

| | |
|---|---|
| **[SMB]({{< relref "/protocols/smb.md" >}})** | NTLMv2. 비밀번호가 네트워크를 건너지 않으며, 공유는 읽는 사람에게 접근 마스크를 통해 시도하기 전에 이미 읽기 전용임을 알려 줍니다. |
| **[WebDAV]({{< relref "/protocols/webdav.md" >}})** | [TLS]({{< relref "/security/tls.md" >}}) 위의 HTTP Basic. 접근 가능한 주소에서 평문은 거부되며, 앞단의 프록시가 TLS를 종료한다고 `plaintext = true`로 지정한 경우는 예외입니다. 사용자가 이용할 수 없는 공유는 403이 아니라 404로 응답하여 존재가 확인되지 않도록 합니다. |
| **[SFTP]({{< relref "/protocols/sftp.md" >}})** | **공개 키** 또는 신뢰하는 기관이 발급한 **SSH 인증서**. 서버는 비밀을 보유하지 않으며, 인증서를 쓰면 사용자의 접근 권한이 다른 곳에서 발급되고 만료됩니다. 비밀번호는 없습니다. 비밀번호를 묻는 클라이언트는 키가 피하려고 존재하는 바로 그 일을 하고 있는 것입니다. |
| **[OIDC]({{< relref "/protocols/webdav.md" >}})** (WebDAV 위에서) | ID 공급자가 서명한 **베어러 토큰**. [go-authn/oidc](https://github.com/go-authn/oidc)가 서명, 발급자, audience, 만료를 검증합니다. 여기의 다른 프로토콜에는 토큰을 실을 곳이 없습니다. S3는 SigV4로 서명하며 SigV4에는 베어러 토큰을 위한 필드가 없습니다. [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})에서는 공급자의 보증이 인증서로 도착합니다. |
| **[S3]({{< relref "/protocols/s3.md" >}})** | 헤더 방식 또는 사전 서명(presigned) **SigV4**. 비밀은 HMAC을 계산하여 스스로를 증명하고 네트워크를 건너지 않으므로, NTLMv2와 마찬가지로 디렉터리가 비밀번호를 단순히 확인하는 데 그치지 않고 **보유해야** 합니다. 공유는 버킷입니다. |
| **[NFSv3]({{< relref "/protocols/nfs.md" >}})** | 단독으로는 **아무것도 없습니다**. `AUTH_UNIX`는 주장일 뿐이며, 클라이언트가 "uid 501"이라고 말하면 네트워크는 이를 반박할 수 없습니다. `kerberos` 블록이 이를 해소하며, [인증서 기반 ID]({{< relref "/security/nfs-certificates.md" >}})도 마찬가지입니다. [TLS]({{< relref "/security/tls.md" >}})는 암호화하며 사용자가 아니라 머신을 증명합니다. |

## 결론, 한 번만 말하면 {#the-consequence-stated-once}

**사용 가능한 사용자를 지정한 공유는 NFS로 내보내지 않습니다.** Kerberos를
구성했거나 NFS가 [인증서 기반 ID]({{< relref "/security/nfs-certificates.md" >}})를
사용하는 경우는 예외입니다.

경고도 선택 사항도 아닙니다. *photos는 alice의 것*이라고 말하는 구성과 연결하는
누구에게나 photos를 내주는 프로토콜은 둘 다 충족될 수 없으며, 접근을 조용히
넓히는 쪽이 두 실패 중 더 나쁩니다. 거부는 시작할 때와
[`check`]({{< relref "/configuration/check.md" >}})에서 이유와 함께 출력됩니다.

## 각 프로토콜이 디렉터리에 요구하는 것 {#what-each-one-needs-from-a-directory}

어떤 프로토콜이 특정 사용자에게 서비스할 수 있는지는 구성만으로가 아니라 그
사용자의 소스가 증명할 수 있는 것으로 결정됩니다.
[소스가 증명할 수 있는 것]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}})을
참고하십시오.

## 암호화되는 것 {#what-is-encrypted}

WebDAV, S3, NFS는 `tls = true`를 받으며, SMB 3은 자체 키로 암호화하고 SFTP는
SSH입니다. [TLS와 ACME 인증서]({{< relref "/security/tls.md" >}})를 참고하십시오.
