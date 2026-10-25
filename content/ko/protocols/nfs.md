---
title: "NFS — Kerberos나 인증서가 없으면 아무것도 없음"
linkTitle: "NFS: Kerberos나 인증서가 없으면 아무것도 없음"
weight: 50
description: "NFSv3는 단독으로는 누구도 인증하지 않습니다. Kerberos 또는 클라이언트 인증서로 제한된 공유를 제공하는 방법과 TLS가 더하는 것."
tags: [프로토콜, nfs, kerberos]
---

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

{{< callout type="error" >}}
**NFSv3는 단독으로는 누구도 인증하지 않습니다**

`AUTH_UNIX`는 **주장**입니다. 클라이언트가 "uid 501"이라고 말하면 네트워크는
이를 반박할 수 없습니다. 암호화도 없습니다.
{{< /callout >}}

그래서 **사용 가능한 사용자를 지정한 공유는 NFS로 내보내지 않습니다**. 거부는
시작할 때와 [`check`]({{< relref "/configuration/check.md" >}})에서 이유와 함께
출력됩니다.

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

`allow`도 `writers`도 없는 공유, 즉 연결하는 누구나 읽고 쓸 수 있는 공유는 NFS로
내보내집니다. 프로토콜이 지키지 못할 규칙이 거기에는 없기 때문입니다.

## 예전에는 영구적이었던 거부 {#the-refusal-that-used-to-be-permanent}

`kerberos` 블록은 NFS가 사용자를 구별할 수 있게 하며, 그러면 제한된 공유도 다른
곳과 마찬가지로 NFS로 제공됩니다.

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5`는 클라이언트가 주장하는 uid가 아니라 **티켓이 증명하는** 프린시펄을
운반합니다.

{{< callout type="info" >}}
**`@` 앞의 이름만이 아니라 렐름도 비교합니다**

두 렐름에 각각 `alice`가 있을 수 있으며, 그중 하나만 여러분의 것입니다.
{{< /callout >}}

## TLS 위에서: 사용자가 아니라 머신 {#over-tls-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

v0.9.0부터 NFS는 `tls = true`와 [`tls` 블록]({{< relref "/security/tls.md" >}})으로
RPC-with-TLS(RFC 9289) 위에서 제공됩니다. 암호화를 하며, `client_ca_file`이 있으면
클라이언트가 그 기관의 인증서를 제시하게 하여 어느 **호스트**가 마운트하는지
알 수 있습니다. 그 안의 uid는 여전히 `AUTH_SYS`가 주장하는 값이므로, 제한된
공유는 이 위에서도 **여전히 거부됩니다**. TLS는 제공될 뿐 필수는 아닙니다. 이를
요청하지 않는 클라이언트에게도 열린 공유는 계속 제공됩니다.

## 인증서로 확인되는 사용자 {#a-person-from-their-certificate}

v0.12.0부터 인증서는 **사용자**를 지정할 수 있습니다. go-authn/bridge가 ID 공급자
로그인 후 발급하는 것과 같은 인증서이며, 그러면 사용자를 지정하는 공유가
kerberos 없이 NFS로 제공됩니다.

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file: one is required
}
```

{{< callout type="error" >}}
**Linux 클라이언트의 인증서는 마운트에 속합니다**

해당 마운트의 모든 사용자는 인증서가 지정하는 사용자로 동작합니다. 한 사람이
사용하는 워크스테이션에서는 맞지만, 여러 사람이 로그인하는 머신에서는
틀립니다. 그런 곳에서는 `sec=krb5`를 사용하십시오.
{{< /callout >}}

인증서가 담는 것, 필수인 CRL, 실제 Linux 클라이언트로 측정한 내용(평문 `MNT`,
`tlshd`의 함정)은
[인증서 기반 ID를 쓰는 NFS]({{< relref "/security/nfs-certificates.md" >}})에 있습니다.
