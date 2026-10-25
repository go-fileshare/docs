---
title: "NFS — Kerberos か証明書がなければ何もなし"
linkTitle: "NFS: Kerberos か証明書がなければ何もなし"
weight: 50
description: "NFSv3 は単独では誰も認証しない。Kerberos やクライアント証明書によって制限付き共有を提供できる仕組みと、TLS が加えるもの。"
tags: [プロトコル, nfs, kerberos]
---

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

{{< callout type="error" >}}
**NFSv3 は単独では誰も認証しません**

`AUTH_UNIX` は**申告**です。クライアントが「uid 501」と言えば、ネットワーク上でそれに反論する手段はありません。
暗号化もありません。
{{< /callout >}}

したがって、**利用できる人を指定した共有は NFS ではエクスポートされません**。拒否は起動時と
[`check`]({{< relref "/configuration/check.md" >}})で、理由とともに出力されます。

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

`allow` も `writers` もない共有(接続した誰もが読み書きできる)は NFS でエクスポートされます。
プロトコルが守れないルールが、そこにはないためです。

## かつては恒久的だった拒否 {#the-refusal-that-used-to-be-permanent}

`kerberos` ブロックがあると、NFS は人を区別できるようになり、制限付き共有もほかの場所と同様に
NFS で提供されます。

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5` が運ぶのは、クライアントが主張する uid ではなく、**チケットが証明する**プリンシパルです。

{{< callout type="info" >}}
**比較されるのは `@` の前の名前だけでなく、レルムも含まれます**

2 つのレルムにそれぞれ `alice` がいることがあり、自分のものはそのうちの 1 つだけです。
{{< /callout >}}

## TLS 経由: 人ではなくマシン {#over-tls-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

v0.9.0 以降、`tls = true` と [`tls` ブロック]({{< relref "/security/tls.md" >}})を指定すると、
NFS は RPC-with-TLS(RFC 9289)で提供されます。暗号化を行い、`client_ca_file` を指定すると、
その認証局の証明書をクライアントに提示させます。これは、どの**ホスト**がマウントしているかを示します。
内側の uid は依然として `AUTH_SYS` が主張するものなので、制限付き共有はこの経路でも
**依然として拒否されます**。TLS は提供されるだけで必須ではなく、TLS を要求しないクライアントには、
引き続き開放された共有が提供されます。

## 証明書から得られる人 {#a-person-from-their-certificate}

v0.12.0 以降、証明書は**人**を名指しできます。go-authn/bridge が ID プロバイダーでのログイン後に
発行するものがそうです。この場合、人を指定した共有が kerberos なしで NFS で提供されます。

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file: one is required
}
```

{{< callout type="error" >}}
**Linux クライアントの証明書はマウントに属します**

そのマウントのすべてのユーザーが、証明書が名指しする人として動作します。一人が使うワークステーション
では正しいのですが、複数人がログインするマシンでは誤りです。そこでは `sec=krb5` を使ってください。
{{< /callout >}}

証明書が運ぶもの、必須となる CRL、そして実際の Linux クライアントで計測したこと(平文の `MNT`、
`tlshd` の落とし穴)は、[証明書による ID を使う NFS]({{< relref "/security/nfs-certificates.md" >}})にあります。
