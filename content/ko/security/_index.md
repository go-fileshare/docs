---
title: "무엇이 무엇을 폐기하는가"
linkTitle: "보안"
weight: 40
description: "fileshare가 수락하는 각 자격 증명을 회수하는 메커니즘과, 그 모두가 안전하게 차단되는(fail closed) 이유."
tags: [보안, 폐기]
---

자격 증명은 제시되는 시점에 검사됩니다. 이후에 사용자를 회수하는 것(퇴사한
직원, go-authn/bridge에서 비활성화된 계정, 통째로 꺼진 IdP)은 그에게 발급된
**모든** 자격 증명과 그것으로 이미 열린 세션에 미쳐야 합니다. 여기의 자격 증명은
종류마다 발급되고 검사되는 방식이 다르므로, 각각 다른 메커니즘으로 회수합니다.

| 자격 증명 | 프로토콜 | 회수하는 것 | 추가된 버전 |
|---|---|---|---|
| `users` 디렉터리(SQL, LDAP)의 비밀번호, NT 해시, 애플리케이션 비밀번호 | SMB, WebDAV, S3 | 행 삭제 후 [디렉터리 다시 불러오기]({{< relref "/administration/reload.md" >}}): 새 세대가 시작되고 그 세대가 열었던 연결이 닫힘 | v0.10.0 |
| 공유 권한 부여 | 모든 프로토콜 | [관리 API]({{< relref "/administration/_index.md" >}})의 `Revoke` 또는 `DisableShare`: 새 세대가 시작되고 연결이 닫힘 | v0.8.0 |
| 공급자의 CA가 서명한 SSH 인증서 | SFTP | 해당 [KRL]({{< relref "/security/revocation-lists.md" >}}), 로그인 시와 **모든 작업마다** 검사. 그리고 [공유 신호]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |
| OpenPubkey(opkssh) 인증서 | SFTP | 어떤 목록도 지정할 수 없음. [공유 신호]({{< relref "/security/shared-signals.md" >}})와 `opkssh_max_age` | v0.13.0 |
| 액세스 토큰 | WebDAV | 어떤 목록도 지정할 수 없음. [공유 신호]({{< relref "/security/shared-signals.md" >}}) | v0.13.0 |
| 사용자를 지정하는 X.509 클라이언트 인증서 | NFS | 해당 [CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}), 다음 호출 시. 그리고 [공유 신호]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |

세 번째 열의 메커니즘이 없으면 자격 증명은 만료될 때까지 유효하며, 인증서나
토큰의 경우 그 기간이 곧 수명입니다.

## 그 모두가 안전하게 차단됩니다(fail closed) {#every-one-of-them-fails-closed}

가져올 수 없는 폐기 목록, `max_age`보다 오래된 사본, 자체 `NextUpdate`가 지난 CRL,
`max_age` 안에 응답이 없는 공유 신호 전송자. 이 중 어느 것도 "폐기된 것이 없음"으로
해석되지 않습니다. "이 서버는 알지 못함"이며, 그 메커니즘이 관할하는 자격 증명은
다시 알게 될 때까지 **거부됩니다**.

{{< callout type="error" >}}
**대가는 가용성이며, 의도적으로 치르는 것입니다**

목록에 접근할 수 없을 때 모든 것을 통과시키는 폐기 검사는 목록을 막을 수 있는
공격자에게 무력화되는 검사입니다. 브라우저의 soft-fail OCSP가 "충돌하면 끊어지는
안전벨트"라고 불린 이유입니다. 그러므로 목록이나 전송자는 그것이 관할하는
로그인만큼 안정적으로 제공되어야 하며, 목록이 낡아 가고 있음을 알려 주는 메트릭이
[알림을 설정할 대상]({{< relref "/administration/health.md#what-to-alert-on" >}})입니다.
{{< /callout >}}

사본은 파싱에 성공한 목록으로만 교체됩니다. 손상되어 도착한 목록은 마지막 정상
목록을 그대로 두며, 그 목록은 **그것을** 가져온 시점부터 `max_age`에 계속
산입됩니다. 또한 목록은 **HTTPS로만** 가져옵니다. 경로상의 공격자가 바꿀 수 있는
목록은 아무것도 폐기하지 못하므로, 평문 HTTP URL은 시작할 때 거부됩니다.

## 전송 {#transport}

자격 증명 자체는 네트워크를 평문으로 건너서는 안 됩니다. 그것이
[TLS]({{< relref "/security/tls.md" >}})입니다. TLS 위의 WebDAV, S3, NFS, 파일 또는
ACME에서 가져온 인증서, 그리고 v0.9.0부터는 **비밀번호를 쓰는 WebDAV의 평문
거부**입니다.
