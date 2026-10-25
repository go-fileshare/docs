---
title: "check, 사람들이 사용 중인 것을 재시작하기 전에"
linkTitle: "check, 재시작하기 전에"
weight: 30
description: "fileshare check가 제공되기 전의 구성에 대해 출력하는 내용과 그 읽는 방법."
tags: [구성, check]
---

```
$ fileshare check /etc/fileshare.d
ATTIC

SHARE    IMAGE             FILESYSTEM  WHO MAY CONNECT           WHO MAY WRITE             SMB  WEBDAV  NFS  SFTP
photos   /srv/photos.img   fat32       alice and bob             alice                     yes  yes     NO   yes
scratch  /srv/scratch.img  ext4        anyone who authenticates  anyone who authenticates  yes  yes     yes  yes

photos is not served over nfs: it is restricted to alice and bob, and NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client makes about itself and the wire cannot disagree with it. A kerberos block lifts this: sec=krb5 carries a principal a ticket proves; so does identity = "certificate" on the nfs serve block: RPC-over-TLS with a client certificate naming the person

USER   FROM                      AUTHENTICATES WITH                         SMB  WEBDAV  SFTP
alice  the configuration file    a password and 1 key from /etc/…/alice.pw  yes  yes     yes
bob    the configuration file    a password from /etc/…/bob.pw              yes  yes     -
dora   ldaps://ldap.example.org  a password check and an NT hash            yes  yes     -
eli    ldaps://ldap.example.org  a password check                           -    yes     -

this configuration can be served
```

모든 이미지는 서버가 여는 방식 그대로 열렸다가 다시 닫히며, 아무것도 제공되지
않습니다. `read_only = true`를 지정하지 않은 공유는 **읽기-쓰기**로 열립니다.
항상 읽기 전용으로, 그리고 배타적으로 열리는 것은 장치뿐이므로, 실행 중인 서버가
잡고 있는 장치는 여기서 사용 중이라는 이유로 거부될 수 있습니다.

**자격 증명이 어디에서 오는지는 출력하지만, 그 내용은 결코 출력하지 않습니다.**
사용자별 열이 있는 것은
[`Identity.Can`](https://github.com/go-authn/directory)이 존재하기 때문입니다.
eli는 dora와 같은 그룹이지만 SMB로는 여전히 제공할 수 없습니다. LDAP이 그의
비밀번호를 보유하고 있으면서 누구에게도 내주지 않기 때문입니다.

## 열 읽기 {#reading-the-columns}

| | |
|---|---|
| 별표가 붙은 파일 시스템 | 드라이버가 인식된 것이 아니라 공유에 의해 **지정된** 것입니다 — [공유]({{< relref "/configuration/shares.md" >}}) 참고 |
| 프로토콜 아래의 `NO` | 그 공유는 해당 프로토콜로 내보내지 않으며, 이유는 표 아래에 출력됩니다 |
| 공유별 프로토콜 아래의 `-` | 공유의 `protocols`가 해당 프로토콜을 제외합니다 |
| 사용자별 프로토콜 아래의 `-` | 그 사용자의 소스가 해당 프로토콜에 필요한 것을 증명할 수 없습니다 — [ID]({{< relref "/configuration/identity.md" >}}) 참고 |

## 그 밖에 알려 주는 것 {#what-else-it-says}

표 다음에, 해당되는 경우 `check`는 구성을 신뢰하기 전에 알아야 할 것을
알려 줍니다.

- `oidc` 블록이 있으면, 그 토큰이 WebDAV에서만 수락된다는 점과 공급자의 이름이
  로컬 이름(`local_names`)인지 여부 —
  [ID]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}) 참고
- 프로토콜별로 무엇이 **암호화**되고 무엇이 **의도적으로 평문**인지 —
  [TLS]({{< relref "/security/tls.md#what-check-says" >}}) 참고
- **폐기 목록**이 어디에서 오는지, 그리고 가져올 수 없는 동안 어떤 일이
  일어나는지 — [무엇이 무엇을 폐기하는가]({{< relref "/security/_index.md" >}}) 참고
- 관리 API의 `DisableShare`로 **오프라인 처리되어** 제공되지 않는 공유 —
  [관리 API]({{< relref "/administration/_index.md#disabling-a-share" >}}) 참고

KRL, 인증서 기반 NFS ID, 공유 신호가 있는 서버에서는 해당 부분이 다음과 같이
표시됩니다.

```
sftp: the provider's SSH certificates are checked against the KRL at https://bridge.example.org/ssh/krl, at login and on every operation; while it is unknown or older than 1h0m0s they are refused

federated people: revocations come from the shared signals transmitter https://bridge.example.org (CAEP session-revoked, polled); whatever the provider issued them before a revocation is refused -- tokens, SSH and OpenPubkey certificates, NFS certificates -- and open sessions stop. Refused while it is not heard from within 10m0s; revocations kept 192h0m0s in /var/lib/fileshare/revocations.json

nfs: identities come from client certificates signed by /etc/fileshare/bridge-x509-ca.pem, revoked by the CRL at https://bridge.example.org/x509/crl (refused while it is unknown or older than 1h0m0s).
     ⛔ a Linux client's certificate belongs to a MOUNT: every user of that mount is the person it names
```

`check`는 목록을 가져오지 않습니다. 목록이 어디에서 오는지와 목록이 없을 때
어떤 일이 일어나는지를 알려 줄 뿐입니다. 반면 ID 공급자의 디스커버리 문서와
키는 **읽습니다**. 발급자에 접근할 수 없는 `oidc` 블록은 `check`가
거부합니다.
