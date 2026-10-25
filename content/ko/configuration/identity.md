---
title: "사용자, 그룹, 디렉터리"
weight: 20
description: "사용자가 어디에서 오는지(user 및 group 블록, SQL, LDAP, ID 공급자)와 각 소스가 사용자에게 허용하는 것."
tags: [구성, ID, oidc, ldap, sql]
---

`user` 블록은 한 가정을 위한 디렉터리 전체입니다. 이미 데이터베이스나 LDAP에
사용자가 있는 사이트는 그들을 낡아 버릴 두 번째 장소로 복사해서는 안 되므로,
`users` 블록이 사용자가 있는 곳에서 직접 읽습니다.

```hcl
users "sql" {
  driver   = "postgres"                 # or sqlite, or mysql
  dsn_file = "/etc/fileshare/dsn"       # a DSN holds a password: it lives in a file
  users    = "select login, null, nt_hash, ssh_keys from staff"
  groups   = "select team, member from team_members"
}

users "ldap" {
  url                = "ldaps://ldap.example.org"
  base_dn            = "ou=people,dc=example,dc=org"
  bind_dn            = "cn=reader,dc=example,dc=org"
  bind_password_file = "/etc/fileshare/bind.pw"
}
```

**쿼리는 사용자가 작성합니다.** 사이트의 사용자는 이미 그 사이트의 형태로
존재하며, 여기서 스키마를 새로 만들면 사용자를 두 번째 스키마로 복사해야 하기
때문입니다. 사용자 쿼리의 열은 이름이 아니라 **위치로** 읽습니다. 이름, 그다음
비밀번호, NT 해시, SSH 키(한 줄에 하나), TOTP 비밀 중 필요한 것이 이 순서로
옵니다. NULL은 그 사용자가 가지지 않은 자격 증명을 뜻하며, 위의 예제가 비밀번호
자리에 `null`을 선택하는 이유가 이것입니다. 비밀번호 열은 비밀번호 그 자체로
취급되며, 해시된 비밀번호를 위한 설정은 여기에 없습니다.

LDAP 쪽은 Samba를 인식하는 디렉터리가 이미 게시하는 `sambaNTPassword`,
`sshPublicKey`, `memberUid`를 읽습니다. 사용자와 그룹이 어디에 있고 어떤 속성이
그들을 지칭하는지는 설정할 수 있습니다. `user_filter`
(기본값 `(|(objectClass=posixAccount)(objectClass=account)(objectClass=uidObject))`),
`user_attribute`(`uid`), `group_base_dn`(`base_dn`), `group_filter`
(`(|(objectClass=posixGroup)(objectClass=groupOfNames))`), `group_attribute`
(`cn`), `group_member_attribute`(기본값은 `memberUid`와 `member` 둘 다),
그리고 기본값이 없는 `totp_attribute`입니다. `sambaNTPassword`와
`sshPublicKey`는 그 이름으로 읽습니다. `member` 값은 DN이며, 사용자 검색이
반환한 사용자의 DN일 때만 인정됩니다. 트리의 다른 곳에 있는 `uid=alice`는
alice가 아닙니다. 이 기본값들(v0.30.0부터)은 go-authn/authnd v0.11.0이 게시하는
것을 읽습니다. 이전에는 `(objectClass=posixAccount)`와 `(objectClass=posixGroup)`뿐이었으며,
기관 자체의 디렉터리에 대해서는 이제 더 많은 항목을 읽습니다.

{{< callout type="error" >}}
**다른 머신으로의 평문 바인드 금지 (v0.19.0부터)**

`users "ldap"` 블록은 `start_tls = true`가 아니면 다른 머신으로의 `ldap://`를
거부합니다. 바인드가 확인하는 모든 비밀번호가 평문으로 네트워크를 건너게 되기
때문입니다. `ldaps://`는 허용되며, 중간에 아무도 없는 루프백 주소로의 `ldap://`와
`ldapi://`도 허용됩니다. 이를 끌 수 있는 스위치는 없습니다(go-authn/directory
v0.10.0).
{{< /callout >}}

## 순서, 그리고 하나의 예외 {#order-and-the-one-exception}

소스는 **작성된 순서대로** 조회되며, 이름을 아는 첫 번째 소스가 그 이름을
소유합니다. `user`와 `group` 블록이 먼저 오므로, 로컬에 적어 둔 서비스 계정이
LDAP의 같은 이름 사용자에게 덮어써지지 않습니다.

**그룹은 예외입니다.** 그룹의 구성원은 모든 소스의 **합집합**입니다. 한 팀의
사용자가 파일에도 있고 데이터베이스에도 있을 수 있기 때문입니다.

```hcl
share "photos" {
  allow   = ["@engineers", "alice"]
  writers = ["@owners"]
}
```

{{< callout type="info" >}}
**시작할 때 확장되고, 다시 불러올 때마다 다시 확장됩니다**

v0.10.0부터 `users` 블록(SQL, LDAP)은 `reload = "5m"`마다, `SIGHUP` 수신 시,
관리 API의 `ReloadDirectory` 호출 시 다시 읽습니다. LDAP에서 바뀐 구성원은 더
이상 재시작을 기다리지 않습니다. 다시 불러오기가 제자리에서 적용하는 것과
재시작하는 것, 그리고 비워진 그룹이 그 공유를 모든 사람에게 열지 **않는**
이유는 [디렉터리 다시 읽기]({{< relref "/administration/reload.md" >}})에 있습니다.
디렉터리는 여전히 연결마다 조회되지 않습니다. 그것은 다른 설계이며, 이것은
그 설계가 아닙니다.
{{< /callout >}}

## 소스가 증명할 수 있는 것과 각 프로토콜이 필요로 하는 것 {#what-a-source-can-prove-and-what-each-protocol-needs}

사이트가 마운트할 때 비로소 알게 되는 부분이므로, [`check`]({{< relref "/configuration/check.md" >}})가
가장 먼저 알려 줍니다.

| 소스가 가진 것 | SMB | S3 | WebDAV | SFTP |
|---|---|---|---|---|
| 비밀번호(파일 또는 평문 열) | yes | yes | yes | — |
| NT 해시(`sambaNTPassword`, `nt_hash`) | yes | **no** | — | — |
| 바인드만 있음(LDAP) | **no** | **no** | yes | — |
| 공개 키 또는 신뢰하는 CA | — | — | — | yes |

{{< callout type="error" >}}
**NTLMv2에는 비밀번호나 그 MD4가 필요하며, 다른 것으로는 안 됩니다**

클라이언트는 SMB 서버에 비밀번호를 보내지 않고 그로부터 계산한 증명을
보냅니다. 따라서 비밀번호를 *확인*하기만 하는 디렉터리는 확인이 아무리 훌륭해도
SMB에 응답할 수 없습니다. 이는 이 프로그램의 한계가 아니라 프로토콜의
속성이며, 어떤 구성으로도 바꿀 수 없습니다. WebDAV는 "이 비밀번호가 맞는가"만
묻고, 이는 바인드로 답할 수 있습니다.
{{< /callout >}}

디렉터리가 이름은 지정하지만 아무것도 증명하지 못하는 사용자는 정당한 사용자입니다.
비밀 정보가 다른 곳에 있는 목록일 뿐이며, `check`는 이를 사용자가 직접 알아내게
두지 않고 한 줄로 알려 줍니다.

## 이 파일이 아니라 ID 공급자가 지정하는 사용자 {#people-the-identity-provider-names-not-this-file}

연합(예: [go-authn/bridge](https://github.com/go-authn/bridge)를 통한 RENATER)은
프로젝트에 누가 있는지 알고 있으며, 공유는 목록을 복사하는 대신 **연합에**
물을 수 있습니다. [`oidc` 블록]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}})이
있으면 다음과 같이 합니다.

```hcl
share "photos" {
  image   = "/srv/photos.img"
  allow   = ["oidc:groups:urn:mace:univ-example.fr:photos", "oidc:user:bob@univ-example.fr", "alice"]
  writers = ["oidc:groups:urn:mace:univ-example.fr:photos"]
}
```

`oidc:groups:<value>`는 토큰의 groups 클레임에 해당 값이 있는 사용자이고,
`oidc:user:<name>`은 토큰이 이름으로 지정한 사용자입니다. 표기는 opkssh의 것이어서,
하나의 어휘로 누가 셸에 접근하고 누가 공유에 접근하는지를 표현합니다. 규칙이
지정한 사용자는 공급자가 보증하는 범위에서 이 서버에 알려진 사용자(열린
공유 포함)이며, 어떤 규칙도 지정하지 않은 사용자는 여전히 낯선 사람입니다. 같은
규칙이 [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})에도
적용되며, 이 경우 그룹은 토큰이 아니라 인증서에 담겨 전달됩니다.

{{< callout type="error" >}}
**규칙은 공급자의 사용자에 대해서만 적용됩니다**

`oidc:user:bob`은 공급자가 보증하는 bob입니다. 비밀번호가 있는 bob이라는 로컬
계정은 다른 사람이며 어떤 규칙에도 일치하지 않습니다. `oidc` 블록이 없는 규칙,
형식이 잘못된 규칙, 접속은 허용되지 않으면서 쓰기가 허용되는 규칙은 **시작할 때
거부됩니다**.
{{< /callout >}}

{{< callout type="error" >}}
**공급자의 `alice`는 명시하지 않는 한 로컬 `alice`가 아닙니다 (v0.20.0부터)**

`allow`나 `writers`의 일반 이름은 로컬 계정입니다. 토큰이나 공급자 인증서는
공유의 `oidc:` 규칙으로만 도달합니다. 공급자의 이름이 곧 로컬 이름인 사이트는
`oidc` 블록에 `local_names = true`를 쓰며, 이때 `domains`를 함께 설정하거나
공급자가 통제하는 클레임에서 이름을 가져와야 합니다. 사용자가 스스로
`preferred_username`을 고르는 공급자라면, 그렇지 않을 경우 가입하는 누구에게나
로컬 계정의 공유(쓰기 포함)가 넘어가게 됩니다. v0.20.0 이전에는 이것이
기본값이었습니다(보안 감사 F4). `local_names`가 없어 거부된 연합 이름은 추가할
행과 함께 로그에 기록되며, `fileshare check`는 어떤 규칙이 적용 중인지
알려 줍니다.

NFS 클라이언트 인증서(`identity = "certificate"`)는 예외입니다. 그 CA는 바로
그 목적으로 `client_ca_file`이 고정한 것이고, 일반 이름이 NFS 공유가 사용자를
지정할 수 있는 유일한 방법이므로, `local_names`가 무엇이든 그 이름은 로컬
이름입니다.
{{< /callout >}}

### 어느 기관인가 {#which-institutions}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
  domains  = ["univ-a.fr", "univ-b.fr"]   # nobody else from the federation gets in
}

share "projet-x" {
  image   = "/srv/projet-x.img"
  allow   = ["oidc:groups:urn:mace:univ-a.fr:projet-x", "oidc:domain:univ-b.fr"]
  writers = ["oidc:groups:urn:mace:univ-a.fr:projet-x"]
}
```

`domains`는 SFTP와 WebDAV 모두에서, 그리고 NFS 클라이언트 인증서에 대해
인증할 때 검사됩니다. 이름은 `<무언가>@<그중 하나>` 형식이어야 하며 전체를
비교합니다(`evilunivb.fr`은 `univb.fr`이 아닙니다). `oidc:domain:`은 공유 하나에
대한 같은 검사입니다. 도메인은 공급자가 보증하는 범위까지 신뢰할 수 있습니다.
go-authn/bridge는 IdP의 연합 메타데이터가 해당 범위(scope)를 부여하지 않은
eppn이나 subject-id를 버립니다.

**그룹**은 기관의 IdP가 릴리스하는 값으로, go-authn/bridge의
`claims { groups = [...] }`가 `groups` 클레임으로 변환합니다. 기본값은
`eduPersonEntitlement`(연구실이나 VO의 그룹)이며, 또는
`eduPersonScopedAffiliation`(`staff@univ-a.fr`, `student@univ-b.fr`)입니다.
