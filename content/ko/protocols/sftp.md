---
title: "SFTP — 키 또는 인증서"
linkTitle: "SFTP: 키 또는 인증서"
weight: 30
description: "공개 키 또는 SSH 인증서를 쓰는 SFTP: 로컬 및 공급자 기관, OpenPubkey, 도메인 부여, 소스 주소, 폐기."
tags: [프로토콜, sftp, ssh]
---

모든 클라이언트에 이미 있습니다. `sftp`는 OpenSSH와 함께 제공되고, Finder와
GNOME이 마운트하며, 편집기들도 사용합니다. 그러나 여기서 형태가 맞지 않는 유일한
프로토콜이기도 합니다. 사용자는 공유 목록이 아니라 **하나의** 파일 시스템에
로그인하여 들어가기 때문입니다.

그래서 공유는 방금 인증한 사용자를 위해 만들어진 트리의 최상위 디렉터리가
됩니다.

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

alice가 이용할 수 없는 공유는 **alice가 볼 수 있는 디렉터리가 아니며**, 읽기만 할
수 있는 공유는 쓰기를 허용했을 드라이버보다 앞선 *트리에서* 쓰기를 거부합니다.

{{< callout type="info" >}}
**두 공유에 걸친 이름 변경은 거부됩니다**

그것은 두 이미지에 걸친 복사와 삭제가 될 것이며, 어디서든 이름 변경이 약속하는
바가 아닙니다.
{{< /callout >}}

## 구성 {#configuration}

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # optional

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # nothing here: her certificate is her credential
```

`trusted_user_ca_file`이 있으면, 그 기관이 서명하고 프린시펄 중에 해당 사용자를
지정하는 인증서만으로 충분합니다. 따라서 사용자의 접근 권한은 **다른 곳에서
발급되고 만료되며**, 누군가 들어오거나 떠날 때 여기의 어떤 파일도 편집되지
않습니다.

서명, 유효 기간, 프린시펄은 `x/crypto/ssh`의 `CertChecker`가 검사하며,
**OpenSSH 자체 클라이언트**로 검증되었습니다. OpenSSH 클라이언트도 인증서를 치워
두면 같은 키를 거부합니다.

{{< callout type="warning" >}}
**`host_key_file`이 없으면 시작할 때마다 새 ID가 생성됩니다**

서버가 이를 알리며, 이전에 이 서버를 본 모든 클라이언트는 키가 바뀌었다고
경고합니다. 클라이언트가 제 역할을 하는 것입니다.
{{< /callout >}}

## 이 호스트를 위한 인증서: 도메인 부여 {#certificates-meant-for-this-host-the-domain-grant}

v0.22.0부터 해당됩니다. [EuroHPC Federation Platform의 SSH CA](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-overview/)
(GÉANT / MyAccessID)는 연합의 모든 사이트가 신뢰하는 인증서를 서명합니다. 각
인증서는 `ssh-domain-grant@core.aai.geant.org` 확장에 자신이 어느 호스트를 위한
것인지 기록합니다. sshd는 이 확장을 무시하며, fileshare도 v0.22.0까지는 그랬습니다.
다른 누군가의 머신에 부여된 인증서가 여기서도 똑같이 통했습니다.
`ssh_domains`를 쓰면 [go-authn/sshcert](https://github.com/go-authn/sshcert)가
명세대로 부여를 읽고, 파싱하고, 일치 여부를 확인합니다. 부여가 이 호스트의 이름을
하나도 지정하지 않는 인증서는 거부됩니다.

```hcl
trusted_user_ca_file = "/etc/fileshare/efp-ssh-ca.pub"
ssh_domains          = ["files.example.org", "sftp.example.org"]
# ssh_accept_ungranted = true   # only if a trusted CA never writes a grant
```

| 키 | |
|---|---|
| `ssh_domains` | 부여가 지정하는 방식 그대로의 이 호스트 이름들. 인증서는 부여의 패턴 하나가 이 이름 중 하나와 일치하면 수락됩니다. `*`는 **하나의 레이블 안에서 하나 이상의 문자**입니다. `*.example.org`는 `files.example.org`를 허용하지만 `a.files.example.org`는 허용하지 않으며, `files-*.example.org`는 `files-.example.org`를 허용하지 않습니다. 비교는 대소문자를 구분하지 않습니다. 이들은 패턴이 아닌 호스트 이름이며, 각각 한 번만 나열하고, 적어도 하나의 기관(`trusted_user_ca_file` 또는 `oidc` 블록의 `ssh_ca_file`)을 신뢰해야 합니다. |
| `ssh_accept_ungranted` | 부여가 **없는** 인증서를 받아들입니다. 기본값은 꺼짐이며, `ssh_domains` 없이 지정하면 의미가 없으므로 거부됩니다. |

| 인증서 | `ssh_domains` | `+ ssh_accept_ungranted` | `ssh_domains` 없음 |
|---|---|---|---|
| `ssh_domains` 중 하나에 부여됨 | 허용 | 허용 | 허용 |
| 다른 호스트에만 부여됨, 또는 `[]` | 거부 | 거부 | 허용 |
| 부여 없음 | 거부 | 허용, 기록됨 | 허용 |
| 파싱되지 않는 부여(`null`, `[1]`, compact 아님...) | 거부 | **거부** | 허용 |

`ssh_domains`가 없으면 아무것도 읽지 않으며 아무것도 바뀌지 않습니다. 부여는
**기관**이 서명한 인증서, 곧 `trusted_user_ca_file`의 인증서와 공급자의
인증서(`ssh_ca_file`)에서 읽습니다. OpenPubkey 인증서는 사용자 자신의 키로
서명되므로 그 안의 부여는 사용자가 자신에 대해 하는 말일 뿐입니다. 이 인증서의
audience는 대신 `opkssh_client_id`입니다. 일반 키에는 부여가 없으며 해당 사항이
없습니다.

모든 거부는 인증서의 일련번호, 키 ID, 이유와 함께 기록됩니다.

```
sftp: alice: certificate 42 ("alice@efp") refused: its domain grant [login.example.eu] names none of this host's names [files.example.org sftp.example.org]
sftp: alice: certificate 43 ("staff") refused: it has no domain grant (ssh-domain-grant@core.aai.geant.org), and ssh_domains requires one
```

각 결정은 결과 `granted`, `accepted_ungranted`, `refused_not_granted`,
`refused_absent`, `refused_malformed`와 함께
`fileshare_sftp_domain_grant_total{result}`에도 집계됩니다. 클라이언트는 키를
가지고 있음을 증명하기 전에 인증서를 제시하므로, 이 집계는 사용자가 아니라
시도의 수입니다.

{{< callout type="error" >}}
**fail-closed인 이유**

두 번째 기관을 신뢰하는 호스트를 생각해 보십시오. 직원용 자체 CA, 테스트 CA, 같은
파일에 덧붙인 EFP의 스테이징 CA, 또는 부여가 없는 go-authn/bridge 클라이언트
등입니다. 그 기관의 인증서에는 부여가 없습니다. 부여가 없다는 것이 "제한 없음"을
뜻한다면, 그 호스트에서 이 필터는 아무것도 걸러 내지 못합니다.

여기서는 부여가 없는 인증서는 `ssh_accept_ungranted`가 달리 지정하지 않는 한
거부됩니다. 부여가 있지만 파싱되지 않으면 내용과 관계없이 거부됩니다. 부여를
기록한 기관은 그 인증서를 제한하려는 의도였기 때문입니다. go-authn/sshcert가
[다중 CA 시나리오](https://github.com/go-authn/sshcert#why-fail-closed-the-multi-ca-scenario)를
설명합니다.
{{< /callout >}}

### EFP의 CA를 쓰는 경우 {#with-efps-ca}

`trusted_user_ca_file`은 OpenSSH의 `TrustedUserCAKeys`입니다. EFP의
[신뢰 페이지](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-trust/)에 나온 대로,
EFP가 <https://sshca.my-eurohpc.eu/config>에 게시하는 키를 넣으십시오. 이 URL은
`{"PublicKey": "ssh-ed25519 ..."}`를 반환하며, 파일에는 `PublicKey`의 값이
한 줄로 들어갑니다.

```sh
curl -s https://sshca.my-eurohpc.eu/config | jq -r '.PublicKey' > /etc/fileshare/efp-ssh-ca.pub
```

그런 다음 `ssh_domains`를 호스팅 기관의 부여가 사용하는 이름으로 설정하십시오.
EFP 인증서의 유일한 프린시펄은 MyAccessID 식별자(`<id>@myaccessid.org`)이며,
이것이 로그인에 쓰이는 사용자 이름입니다. 그 이름의 `user` 블록은 자체 자격
증명이 없어도 이를 받습니다.

```hcl
user "u1234@myaccessid.org" {}
```

### go-authn/bridge를 쓰는 경우 {#with-go-authnbridge}

`ssh_certificates = true`와 `ssh_domain_grants = ["files.example.org"]`를 지정한
[go-authn/bridge](https://github.com/go-authn/bridge) 클라이언트(v0.19.0 이상)는
이 호스트에 부여된 인증서를 발급합니다. `ssh_principal_claim = "voperson_id"`로
하면 유일한 프린시펄은 EFP 프로필과 같이 사용자의 `voperson_id`이며, 여기에
작성할 `user` 블록이 바로 그것입니다. bridge의 CA는 이 서버가 다른 CA를 신뢰하는
방식과 같게 신뢰하십시오.

- 로컬 계정용 `trusted_user_ca_file`로. bridge는 `/ssh/config`에서 EFP 형식으로
  키를 제공하므로, 위의 `curl | jq` 행은 bridge의 URL로도 동작합니다.
- 또는 [공급자가 보증하는 사용자](#people-the-identity-provider-vouches-for)용
  `oidc` 블록의 `ssh_ca_file`로.

bridge의 다른 클라이언트는 부여를 기록하지 않습니다. 그 인증서가 여기서 계속
동작해야 한다면 `ssh_accept_ungranted`가 그 용도입니다. 하지만 이는 신뢰하는 모든
기관의 부여 없는 인증서를 **전부** 받아들이므로, 해당 클라이언트에 부여를 지급하는
편이 낫습니다.

### 주소에 고정된 인증서 {#a-certificate-pinned-to-an-address}

bridge의 `ssh_source_address`는 `ssh-keygen -O source-address=...`와 같이
`source-address` 크리티컬 옵션을 기록합니다. 이를 담은 인증서는 `ssh_domains`
유무와 관계없이 **그 주소에서만** 수락됩니다. 인증서가 허용하는 주소에서의
로그인은 허용되며, `ssh_domains`가 설정된 경우 부여가 이 호스트도 지정해야
합니다. 다른 주소에서의 로그인은 거부됩니다.

| fileshare | `trusted_user_ca_file` | `oidc`의 `ssh_ca_file`, 그리고 OpenPubkey |
|---|---|---|
| v0.22.0 이전 | 적용됨 | **모든 주소에서 거부** |
| v0.22.0, `ssh_domains` 없음 | 적용됨 | **모든 주소에서 거부** |
| v0.22.0, `ssh_domains` 있음 | **모든 주소에서 거부** | **모든 주소에서 거부** |
| v0.22.1, `ssh_domains` 유무와 관계없이 | 적용됨 | 적용됨 |

"모든 주소에서 거부"는 fail closed입니다. 인증서의 제한이 완화된 적은 없고
로그인만 잃었습니다. v0.22.1 이전에는 go-filesystems/sftp의 sshd가 fileshare에
판단을 넘기는 인증서의 모든 크리티컬 옵션을 거부했으며, 이는 공급자의 인증서,
OpenPubkey 인증서, 그리고 `ssh_domains` 아래에서는 모든 인증서에 해당합니다.
v0.22.1은 go-filesystems/sftp v0.5.1을 기반으로 하며, 이 sshd는 `source-address`를
받아들이고 모든 인증서 경로에서 연결이 들어오는 주소(IPv4 또는 IPv6)에 대해
이를 적용합니다.

그 밖의 크리티컬 옵션(`force-command`, `verify-required` 등)은 이 서버가 처리하지
않으므로 거부됩니다.

## ID 공급자가 보증하는 사용자 {#people-the-identity-provider-vouches-for}

v0.11.0부터 `oidc` 블록은 SFTP에도 적용됩니다. SSH에는 토큰을 실을 곳이 없으므로
토큰이 아니라 **인증서**로 합니다.

```hcl
oidc {
  issuer           = "https://login.example.org"
  audience         = "fileshare"
  ssh_ca_file      = "/etc/fileshare/bridge-ca.pub"   # go-authn/bridge's ssh_ca
  opkssh_client_id = "opkssh"                         # OpenPubkey logins
  opkssh_max_age   = "24h"                            # 12h, 24h, 48h, 1week
}
```

- **공급자의 SSH CA가 서명한 인증서** — `authn-bridge ssh-cert`가 연합을 통한
  로그인 후에 작성합니다. 프린시펄은 사용자이고 `groups@go-authn.org` 확장이
  그룹이므로,
  [`oidc:groups:` 규칙]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})이
  적용됩니다. 로컬 계정에 관한 인증서를 다루는 `trusted_user_ca_file`은 **아닙니다**.
  로컬 기관의 인증서가 공급자의 그룹을 주장하면 그 인증서가 지정하는 로컬
  계정으로 해석됩니다.
- **OpenPubkey 인증서**: `opkssh login`이 작성하는 것으로, 사용자 자신의 키로
  서명되며 그 키에 커밋하는 ID 토큰을 가집니다. `opkssh verify`가 검사하는
  방식으로 검사합니다.
  [openpubkey](https://github.com/openpubkey/openpubkey) 검증기(공급자가 게시한
  키에 대한 공급자의 서명, nonce 커밋, 클라이언트 ID, 유효 기간)를 거친 다음
  인증서의 키를 토큰의 키와 대조합니다. SSH 사용자 이름은 토큰의 사용자 이름이어야
  하는데, 둘을 매핑할 `auth_id` 파일이 여기에는 없기 때문입니다:
  `sftp alice@univ-example.fr@files.example.org`. `opkssh_client_id`는 다른
  애플리케이션의 것이 아닌 전용 클라이언트 ID여야 합니다. ID 토큰은 사용자가
  로그인하는 모든 서버로 전달되기 때문입니다.

인증서가 담은 그룹을 확인하려면 다음과 같이 합니다.

```sh
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub     # the Extensions section
```

그리고 서버는 연합 로그인이 있을 때마다
`sftp: alice@univ-a.fr, vouched for by the provider, in groups [...]`라고
알립니다.

{{< callout type="error" >}}
**보증받는 것은 허가되는 것이 아닙니다**

공급자가 보증하는 사용자라도 규칙이 지정하거나, `trust_all`이 공급자가 곧
디렉터리라고 말하거나, `local_names = true`가 공급자의 이름이 이 서버의 이름이며
로컬 계정이 그 이름을 가진다고 말하지 않는 한 여기서는 여전히 낯선 사람입니다.
WebDAV에서 토큰이 통과하는 것과 같은 검사입니다. **프린시펄이 없는** 공급자
인증서는 형식 자체의 정의상 *누구에게나* 유효하므로 거부됩니다.
{{< /callout >}}

`-tags noopenpubkey`는 OpenPubkey 검증기를 제외하며(약 2 MB), 이런 빌드에서
`opkssh_client_id`가 있는 구성은 거부됩니다.

## 인증서 회수하기 {#taking-a-certificate-back}

{{< callout type="error" >}}
**인증서는 로그인할 때 검사됩니다**

공급자에서 사용자를 폐기해도 그것만으로는 이미 발급된 인증서가 폐기되지
않습니다. 인증서는 만료될 때까지 SFTP를 열 수 있고, 이미 열린 SFTP 세션도 그때까지
열려 있습니다. 이들은 디렉터리에 없으므로 [다시 불러오기]({{< relref "/administration/reload.md" >}})와
무관합니다. 그 기간은 인증서의 수명입니다. bridge의 `ssh_ca { validity }`(기본값
12h, 최대 168h, IdP 세션의 종료를 넘지 않음)와 여기의 `opkssh_max_age`입니다.
클라이언트의 재로그인이 허용하는 만큼 짧게 유지하십시오. v0.17.0부터는 이미 열린
opkssh 세션도 인증서(사용자가 직접 서명함)가 무엇이라고 말하든 `opkssh_max_age`에
끝납니다.
{{< /callout >}}

그 기간을 닫는 메커니즘은 두 가지입니다.

- 공급자의 **[KRL]({{< relref "/security/revocation-lists.md" >}})**(`ssh_krl_url`):
  폐기된 인증서는 로그인할 때 거부되고, 그 인증서로 연 세션의 모든 작업은 열린
  파일을 포함해 목록을 다시 조회합니다.
- **[공유 신호]({{< relref "/security/shared-signals.md" >}})**(`ssf`): 공급자가 CAEP
  `session-revoked` 이전에 해당 사용자에게 발급한 모든 것이 거부됩니다. 어떤
  목록에도 없는 OpenPubkey 인증서에 닿는 유일한 방법입니다.

둘 다 fail closed입니다.

## 비밀번호 없음 {#no-password}

비밀번호를 묻는 클라이언트는 키가 피하려고 존재하는 바로 그 일을 하고 있는
것입니다.

## 전송 크기와 열린 파일 {#transfer-size-and-open-files}

fileshare는 OpenSSH의 `limits@openssh.com`에 응답하므로(v0.25.0부터) OpenSSH의
`sftp`가 요청당 32 KiB 대신 최대 255 KiB를 읽고 씁니다. OpenSSH 10.3으로 측정한
결과 약 +25%입니다. 한 세션은 동시에 최대 1024개의 열린 파일과 디렉터리를 가질 수
있습니다. 디렉터리 공유에서는 각각이 호스트의 디스크립터이며, 이 상한은 한
사용자가 모두 소진하는 것을 막습니다.
